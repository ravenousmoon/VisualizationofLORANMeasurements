<h1> <b> Лабораторна робота №6. Розробка додатку для візуалізації вимірювань LORAN </b> </h1>

<b>Мета:</b> розробити додаток, який зчитує дані з емульованої вимірювальної частини LORAN, наданої у вигляді Docker image, та відображає положення об'єкта і базових станцій на графіку в декартових координатах.

<p><b>Завдання:</b></p>
<ol>
    <li>
        <p>
            <b>Розробити додаток для відображення положення об'єкта і базових станцій:</b>
        </p>
        <ul>
            <li>Розробити веб-додаток, який підключається до WebSocket сервера та зчитує дані про часи отримання сигналів базовими станціями.</li>
            <li>Використовувати різниці часу прибуття сигналів для розрахунку місцезнаходження об'єкта.</li>
            <li>Відобразити отримані дані на графіку в декартових координатах.</li>
        </ul>
    </li>
    <li>
        <p>
            <b>Обробка та візуалізація даних:</b>
        </p>
        <ul>
            <li>Обробити дані, отримані через WebSocket, і відобразити положення об'єкта і базових станцій на графіку.</li>
            <li>Здійснити розрахунок координат об'єкта за допомогою методу найменших квадратів та градієнтного спуску.</li>
        </ul>
    </li>
    <li>
        <p>
            <b>Налаштування графіка:</b>
        </p>
        <ul>
            <li>Відобразити координати базових станцій та об'єкта у декартових координатах.</li>
            <li>Використати різні кольори або стилі точок для відображення базових станцій та об'єкта.</li>
        </ul>
    </li>
</ol>
<p>Для запуску системи завантажуємо і налаштовуємо емулятор вимірювальної частини LORAN у вигляді Docker image. Спочатку потрібно завантажити Docker image за допомогою команди:</p>
<code>docker pull iperekrestov/university:loran-emulation-service</code>
<p>Після цього можна запустити контейнер, використовуючи команду:</p>
<code>docker run --name loran-emulator -p 4002:4000 iperekrestov/university:loran-emulation-service</code>
<p>Ця команда створює контейнер з ім'ям <b>loran-emulator</b> та відкриває порт <b>4002</b> на хост-машині, який буде відображатися на порт <b>4000</b> всередині контейнера. Це дозволяє підключитися до емульованої вимірювальної частини LORAN.</p>

<p><b>Створення початкових даних:</b> задаються координати трьох станцій-приймачів, які розташовані на площині та представлені масивами <b>stations_x</b> і <b>stations_y</b>. Їхні координати визначаються як <b>[0, 100000, 0]</b> для осі X і <b>[0, 0, 100000]</b> для осі Y, утворюючи трикутник: одна станція знаходиться в початку координат, друга – на 100 км праворуч, а третя – на 100 км вгору. Ці фіксовані точки слугують опорними станціями для аналізу затримок сигналу.</p>

<p><b>Обчислення TDoA:</b> для обчислення різниці часу приходу сигналу (TDoA) створена функція <b>tdoa_error</b>. Вона приймає параметри: координати об’єкта та станцій, відомі значення затримок сигналу і швидкість поширення сигналу. У функції розраховуються відстані від об’єкта до кожної станції, після чого визначаються затримки сигналу між станціями.</p>

<p><b>Алгоритм оптимізації:</b> для визначення точного положення об’єкта застосовується функція <b>custom_least_squares</b>, яка реалізує градієнтний спуск. Алгоритм ітеративно обчислює суму квадратів помилок, використовуючи функцію <b>tdoa_error</b>, і мінімізує її шляхом оновлення координат об’єкта. Для цього обчислюються похідні помилки за X та Y, після чого значення координат поступово змінюються в напрямку зменшення градієнта.</p>

<p><b>WebSocket-з’єднання:</b> дані про часи отримання сигналів станціями передаються через WebSocket-з’єднання. Асинхронна функція <b>receive_data</b> підключається до сервера за допомогою протоколу WebSocket і очікує надходження інформації. Дані, отримані у форматі JSON, містять ідентифікатори станцій і час отримання сигналу. Після отримання трьох сигналів розраховуються затримки між сигналами для пар станцій <b>delta_t12</b> та <b>delta_t13</b>. Отримані значення передаються до функції оптимізації <b>custom_least_squares</b> для обчислення координат об'єкта. Результати оновлюються в реальному часі на графіку. Завдяки асинхронності система може безперервно приймати нові дані, забезпечуючи актуальне відображення положення об'єкта.</p>

<p><b>Графічний інтерфейс:</b> для інтерактивної роботи користувача використовується Dash. Інтерфейс складається з двох частин: панелі управління та графіка.</p>
<p>У панелі користувач може змінювати швидкість об’єкта, вводячи значення в текстове поле, після чого натискає кнопку "Змінити швидкість". Ця дія надсилає POST-запит до сервера з новою швидкістю.</p>
<p>Графік, створений за допомогою бібліотеки Plotly, відображає положення станцій і об'єкта в реальному часі.</p>

<p>Створюємо веб-додаток для візуалізації вимірювань LORAN:</p>

``` python
import numpy as np
import dash
from dash import dcc, html, Input, Output, State
import plotly.graph_objects as go
import asyncio
import websockets
import json
import requests
import threading

# Ініціалізація Dash
app = dash.Dash(__name__)


# Функція помилки TDoA
def tdoa_error(params, x1, y1, x2, y2, x3, y3, delta_t12, delta_t13, c):
    x, y = params
    d1 = np.sqrt((x - x1) ** 2 + (y - y1) ** 2)
    d2 = np.sqrt((x - x2) ** 2 + (y - y2) ** 2)
    d3 = np.sqrt((x - x3) ** 2 + (y - y3) ** 2)

    delta_t12_calc = (d1 - d2) / c
    delta_t13_calc = (d1 - d3) / c

    error1 = delta_t12_calc - delta_t12
    error2 = delta_t13_calc - delta_t13

    return [error1, error2]


# Налаштування параметрів обчислення об'єкта
def custom_least_squares(tdoa_error_func, initial_guess, args, learning_rate=0.01, max_iterations=10000,
                         tolerance=1e-12):
    x, y = initial_guess
    iteration = 0
    prev_loss = float('inf')

    while iteration < max_iterations:
        loss = sum(e ** 2 for e in tdoa_error_func([x, y], *args))

        if abs(prev_loss - loss) < tolerance:
            break

        prev_loss = loss

        delta = 1e-6
        loss_x = sum(e ** 2 for e in tdoa_error_func([x + delta, y], *args))
        grad_x = (loss_x - loss) / delta

        loss_y = sum(e ** 2 for e in tdoa_error_func([x, y + delta], *args))
        grad_y = (loss_y - loss) / delta

        x -= learning_rate * grad_x
        y -= learning_rate * grad_y

        iteration += 1

    return x, y, iteration


# Ініціалізація графіка
stations_x = [0, 100000, 0]
stations_y = [0, 0, 100000]

fig = go.Figure()
fig.add_trace(go.Scatter(
    x=stations_x,
    y=stations_y,
    mode='markers',
    name='Станції',
    marker=dict(size=12, color='blue')
))

fig.add_trace(go.Scatter(
    x=[None],
    y=[None],
    mode='markers',
    name='Об\'єкт',
    marker=dict(size=12, color='red')
))

fig.update_layout(
    title="Візуалізація положення об'єкта",
    xaxis_title="X координата (м)",
    yaxis_title="Y координата (м)",
    height=800,
)


# Функція для зміни швидкості об'єкта
def update_object_speed(speed):
    payload = {"objectSpeed": speed}
    response = requests.post("http://localhost:4002/config", json=payload)
    return response.json()


# Асинхронна функція для отримання даних через WebSocket
async def receive_data():
    uri = "ws://localhost:4002"
    try:
        async with websockets.connect(uri) as websocket:
            received_times = {}

            while True:
                data = await websocket.recv()
                json_data = json.loads(data)

                source_id = json_data['sourceId']
                received_at = json_data['receivedAt']
                received_times[source_id] = received_at

                if len(received_times) == 3:
                    receivedAt1 = received_times.get("source1")
                    receivedAt2 = received_times.get("source2")
                    receivedAt3 = received_times.get("source3")

                    if None in [receivedAt1, receivedAt2, receivedAt3]:
                        continue

                    delta_t12 = ((receivedAt1 - receivedAt2) / 1000) * 10e8
                    delta_t13 = ((receivedAt1 - receivedAt3) / 1000) * 10e8

                    initial_guess = [50000, 50000]
                    c = 3e8 / 10e8
                    x_opt, y_opt, _ = custom_least_squares(
                        tdoa_error,
                        initial_guess,
                        args=(stations_x[0], stations_y[0], stations_x[1], stations_y[1], stations_x[2], stations_y[2],
                              delta_t12, delta_t13, c)
                    )

                    update_plot(x_opt, y_opt)
                    received_times.clear()

    except Exception as e:
        print(f"Помилка з'єднання з WebSocket: {e}")


# Оновлення графіка
def update_plot(receiver_x, receiver_y):
    fig.data[1].x = [receiver_x]
    fig.data[1].y = [receiver_y]


@app.callback(
    Output('live-graph', 'figure'),
    Input('interval-component', 'n_intervals')
)
def update_graph(n):
    return fig


@app.callback(
    Output('speed-response', 'children'),
    Input('submit-speed', 'n_clicks'),
    State('speed-input', 'value')
)
def change_speed(n_clicks, speed):
    if n_clicks is None or speed is None:
        return "Введіть швидкість."
    try:
        speed = float(speed)
        response = update_object_speed(speed)
        return f"Швидкість об'єкта змінена"
    except ValueError:
        return "Введіть допустиме значення швидкості."


# Інтерфейс
app.layout = html.Div([

    # Контейнер для параметрів
    html.Div([
        html.Label("Швидкість об'єкта (км/год):", style={'marginRight': '10px', 'font-family': 'Arial, sans-serif'}),
        dcc.Input(id='speed-input', type='number', style={'marginRight': '10px', 'border': '1px solid #ccc',
                                                          'border-radius': '4px', 'width': '100px', 'padding': '8px'}),
        html.Button('Змінити швидкість', id='submit-speed', n_clicks=0,
                    style={'backgroundColor': '#4CAF50', 'color': 'white', 'border': 'none',
                           'padding': '10px 20px', 'border-radius': '4px', 'cursor': 'pointer',
                           'transition': 'background-color 0.3s'}),
        html.Div(id='speed-response', style={'marginTop': '10px', 'textAlign': 'center', 'font-family': 'Arial, sans-serif'})
    ], style={'backgroundColor': 'white', 'padding': '20px', 'borderRadius': '10px',
              'boxShadow': '0px 4px 8px rgba(0, 0, 0, 0.2)', 'textAlign': 'center', 'marginBottom': '20px'}),

    # Контейнер для графіка
    html.Div([
        dcc.Graph(id='live-graph', animate=True),
        dcc.Interval(id='interval-component', interval=1000, n_intervals=0)
    ], style={'backgroundColor': 'white', 'padding': '20px', 'borderRadius': '10px',
              'boxShadow': '0px 4px 8px rgba(0, 0, 0, 0.2)'})


], style={'backgroundColor': '#e0e0e0', 'padding': '50px'})


def run_websocket():
    asyncio.run(receive_data())


if __name__ == "__main__":
    # Запуск WebSocket в окремому потоці
    thread = threading.Thread(target=run_websocket)
    thread.start()

    # Запуск Dash-додатку
    app.run_server(debug=True)
```

<p>Результат створення додатку:</p>





