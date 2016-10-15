---
layout: post
author: Kévin Maschtaler
title: "WebSocket-based app with Django Channels and React"
date: 2016-10-16 12:00:00
lang: en
tag:
- django
- react
- redux
- websocket
---
For my very first blog post, I chose to share my exploration of
a library that I wanted to try for some time: Django Channels.

I was also curious about how a React application can be plugged to a backend different from a REST API.

Let's see that with a real example: the Game of Life!
<!--more-->

[![Game of Life in action](/blog/content/game-of-life.gif)](http://morganherlocker.com/post/Playing-with-the-Game-of-Life/)

### Game of Life
The [Conway's Game of Life](https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life)
is a game that requires enough computation to delegate this role to a backend.

The game's principle, which needs no player, is to make some cells live through several generations.
There are some rules that apply to them, such as biologic ones:

> - Any live cell with fewer than two live neighbours dies, as if caused by under-population;
- Any live cell with two or three live neighbours lives on to the next generation;
- Any live cell with more than three live neighbours dies, as if by over-population;
- Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

The point of the app will be to manage the cells life and death from a server,
to visualize them from a web client and manage the right to life and death
thanks to a manual activation of the new generations.


### Game generation and WebSocket delivery

#### What is django-channels?

If you never heard of Django, I let you check the [dedicated web site](https://www.djangoproject.com/), it's very well done.
To make it simple, it is a straightforward and complete web framework,
based on an MVC architecture and a system built around a request, a response, some middlewares between,
and a lot of tools to help "developers with deadlines".

[Django Channels](https://github.com/django/channels) is a beta project [recently adopted](https://www.djangoproject.com/weblog/2016/sep/09/channels-adopted-official-django-project/)
as an Official Project by the `Django Software Foundation`.

It is described by its own creators with these few words:

> Channels extends Django to add a new layer that allows two important features:
WebSocket handling, in a way very similar to normal views; Background tasks, running in the same servers as the rest of Django.
It allows other things too, but these are the ones you’ll use to start with.

#### Channels architecture

The configuration of channels is relatively simple, as wanted by the Django's rules.
It introduces a new layer before HTTP routing.

```python
from channels.routing import route
from .consumers import http_consumer, ws_add, ws_disconnect, ws_receive

channel_routing = [
    route('http.request', http_consumer),  # Classic HTTP request handling
    route('websocket.connect', ws_add),
    route('websocket.receive', ws_receive),
    route('websocket.disconnect', ws_disconnect),
]
```

We need to associate a [`consumer`](https://channels.readthedocs.io/en/stable/getting-started.html#first-consumers)
to each event.
In the case of our application, we dispatch the roles of each consumer as following:

- HTTP Request: Nothing but warn an eventual visitor (optional)
- WebSocket Connect: Register the new client which will be averted at each game's state update
- WebSocket Receive: Compute a new life's generation
- WebSocket Disconnect: Unregister the client for updates

Django Channels gives a concept of client `Group` in order to make the code very readable.

```python
import json

from django.http import HttpResponse
from channels import Group
from channels.handler import AsgiHandler

from .game import Board

game = Board(75, 75)

def http_consumer(message):
    response = HttpResponse('Welcome to Game of Life! You should try websockets.')

    for chunk in AsgiHandler.encode_response(response):
        message.reply_channel.send(chunk)

def ws_add(message):
    Group('game').add(message.reply_channel)
    message.reply_channel.send({
        'text': json.dumps(game.current_generation),
    })

def ws_receive(message):
    game.new_generation()
    Group('game').send({
        'text': json.dumps(game.current_generation),
    })

def ws_disconnect(message):
    Group('game').discard(message.reply_channel)
```

Here, the game behavior is to generate a 75*75 grid with default fixture.
`game.current_generation` returns the game matrix as well as the generation number,
and `game.new_generation()` computes a new cell generation from the previous one.

You will note that the `game` variable is in the consumers scope.
Therefore, all clients share the same game's state and are all warned at the same time
when a new generation is delivered.

We now have a game of life computed in the backend!
Let's see how we can handle it from front-end.

### Frontend with React & Redux

#### Game display

The representation of the game is a simple 2D matrix.
Hence, we only have to create a grid which two basic loops.

```js
// Grid.js
const Grid = ({ matrix }) => (
    <div className="grid">
        <div className="cell-container">
            {matrix.map((column, i) =>
                <div key={i} className="column">
                    {column.map((cell, j) => <Cell key={j} {...cell} />)}
                </div>
            )}
        </div>
    </div>
);

// Cell.js
const Cell = ({ state }) => <div className={`cell ${state}`} />;
```

The state can be either `empty` or `alive`.

#### WebSocket handling
The most interesting part.
I tried some ways to imitate the [native WebSocket behavior](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
through a `socket` reducer and a few actions.

[redux-saga](https://github.com/yelouafi/redux-saga) gives powerful tools
to easily code some complex behaviors,
but in the end, `redux` allows to resolve the problem with a simple and elegant solution:

```js
// actions.js
import { createAction } from 'redux-actions';

export const actionsTypes = {
    CONNECTING: 'SOCKET_CONNECTING',
    OPENED: 'SOCKET_OPENED',
    MESSAGED: 'SOCKET_MESSAGED',
    ERROR: 'SOCKET_ERROR',
    CLOSED: 'SOCKET_CLOSED',
};

export default {
    connect: createAction(actionsTypes.CONNECTING),
    open: createAction(actionsTypes.OPENED),
    message: createAction(actionsTypes.MESSAGED),
    error: createAction(actionsTypes.ERROR),
    close: createAction(actionsTypes.CLOSED),
};
```

```js
// life/actions.js
import { createAction } from 'redux-actions';

export const actionsTypes = {
    NEW_STATE: 'GAME_NEW_STATE',
};

export default {
    newState: createAction(actionsTypes.NEW_STATE),
};
```

```js
// reducer.js
import { actionsTypes } from './actions';

const initialState = {
    loading: false,
    connected: false,
    error: null,
    message: null,
};

const reducer = (state = initialState, { type, payload }) => {
    switch (type) {
    case actionsTypes.CONNECTING:
        return {
            ...state,
            loading: true,
        };
    case actionsTypes.OPENED:
        return {
            ...state,
            connected: true,
            loading: false,
        };
    case actionsTypes.MESSAGED:
        return {
            ...state,
            ...payload,
        };
    case actionsTypes.ERROR:
        return {
            ...state,
            ...payload,
            loading: false,
        };
    case actionsTypes.CLOSED:
        return {
            ...state,
            loading: false,
            connected: false,
        };
    default:
        return state;
    }
};

export default reducer;
```

```js
// initSocket.js
import actions from './actions';
import gameActions from '../life/actions';

export default dispatch => (uri) => {
    if (!('WebSocket' in window)) {
        dispatch(actions.error({ error: 'WebSocket is not supported by your browser' }));
        return;
    }

    const socket = new WebSocket(uri);
    dispatch(actions.connect());

    socket.onopen = () => dispatch(actions.open({ instance: socket }));
    socket.onerror = () => dispatch(actions.error({ error: true }));
    socket.onmessage = evt => dispatch(gameActions.newState({ ...JSON.parse(evt.data) }));
    socket.onclose = () => dispatch(actions.close());
};

```

#### Root assembly

Our final application now have all it needs to use the computational power
of the django backend to generate a game of life.

```js
import initSocketFactory from '../socket/initSocket';

class App extends Component {
    componentDidMount() {
        this.props.initSocket('ws://localhost:8000');
    }

    render() {
        const { socket, game, initSocket } = this.props;
        const loading = socket.loading || !game.matrix.length;2

        return (
            <div>
                {loading && <span>Loading...</span>}
                {!loading && <Grid matrix={game.matrix} />}
            </div>
        );
    }
}

const mapDispatchToProps = dispatch => ({
    initSocket: initSocketFactory(dispatch),
});

export default connect(store => store, mapDispatchToProps)(App);
```

Finally, with a bit of [Material UI](http://www.material-ui.com/).

![Game of Life example](/blog/content/game-of-life.png)


### What's next?

The entire project is open-source and available on Github: [https://github.com/Kmaschta/GameOfLife](https://github.com/Kmaschta/GameOfLife)

It deserves some love and I still have to learn a lot.
The [evolution algorithm](https://github.com/Kmaschta/GameOfLife/blob/5deaa437b758457ee550d1024f069f34a24f8d9c/src/backend/life/game/evolution.py)
is not very efficient, we can imagine two generation modes "auto/manually"
or the capacity to start from different fixtures.

To do so, it misses a reliable protocol to manage sent messages.
The [Web Application Messaging Protocol (WAMP)](http://wamp-proto.org/)
is a good candidate and I strongly recommend to take it a look.

I may write a next part if I implement this protocol in my little project.
Let me know if you are interested!
