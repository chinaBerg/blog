# reconnecting-websocket.js源码分析

> version v1.0.1
> 愣锤 2022/02/15

[reconnecting-websocket](https://github.com/joewalnes/reconnecting-websocket)实现了和[Websocket](https://developer.mozilla.org/zh-CN/docs/Web/API/WebSocket/WebSocket)相同的api，但是可以在ws断开连接时自动进行重连。

## 基本使用

- 创建一个ws server服务

```js
const NodeJsWebsocket = require('nodejs-websocket');

var server = NodeJsWebsocket.createServer(function (conn) {
	console.log('find a new connection.')
  conn.on('error', (error) => {
    console.error('[AssistDesign] connect error: ', error);
  });
});

server.listen(3000, () => {
  console.log('[server] server is running at port 3000');
});
```

- 创建一个客户端连接

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
</head>
<body>
  <script src="https://cdn.bootcdn.net/ajax/libs/reconnecting-websocket/1.0.0/reconnecting-websocket.js"></script>
  <script>
    const cancel = document.querySelector('#cancel');
    const url = `ws://localhost:3000`;

    const ws = new ReconnectingWebSocket(url, null, {
      reconnectInterval: 3000,
    });
    ws.onopen = function() {
      console.log('connect successful', Date.now())
    }

    ws.onclose = function() {
      console.log('connect close', Date.now());
    }

    cancel.addEventListener('click', () => {
      console.log('click close', Date.now())
      ws.close();
    })
  </script>
</body>
</html>
```

然后可以让电脑进行休眠查看ws意外断开重连效果。

## 源码分析

该库主要利用自定义事件，在Websocket实例的监听函数中，触发用户设置的监听函数。然后在Websocket实例关闭的钩子中重新进行Websocket的实例化逻辑，以此达到重连的功能。

### 自定义事件

`EventTarget.dispatchEvent(event)`向一个指定的事件目标派发一个事件，并以合适的顺序同步调用目标元素相关的事件处理函数。参数`event`。具体可看[EventTarget.dispatchEvent文档](https://developer.mozilla.org/zh-CN/docs/Web/API/EventTarget/dispatchEvent)、[Event文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Event/Event)。

```js
// 监听自定义事件
window.addEventListener('custom-event', function() {
  console.log('custom event emitted');
}, false);

/**
 * 创建一个新的事件对象
 * @see https://developer.mozilla.org/zh-CN/docs/Web/API/Event/Event
 */
const event = new Event('custom-event', {
  bubbles: true, // 冒泡
  cancelable: false, // 不可取消
});
// 触发自定义事件
window.dispatchEvent(event);
```

触发自定义事件的目标也可以是其他目标:

```js
const div = document.createElement('div');
div.addEventListener('dom-custom-event', function() {
  console.log('dom custom event emitted');
}, false);

const event2 = new Event('dom-custom-event', {
  bubbles: true, // 冒泡
  cancelable: false, // 不可取消
});
div.dispatchEvent(event2);
```

`Event`不兼容**IE**，替代方案是使用`document.createEvent`:

```js
/**
 * 创建自定义事件
 * document.createEvent作用是创建指定类型的事件
 * @see https://developer.mozilla.org/zh-CN/docs/Web/API/Document/createEvent
 */
const ieEvent = document.createEvent('CustomEvent');
// 创建一个新的事件对象
ieEvent.initCustomEvent('ie-custom-event', false, false, {
  // other your custom params
  myParam: 'this is data',
});

// 触发自定义事件
// 控制台输出 ie-custom-event emitted:  this is data
window.dispatchEvent(ieEvent);
```

### 源码实现

源码实现近几百行，加了注释如下：

```js
(function (global, factory) {
    if (typeof define === 'function' && define.amd) {
        define([], factory);
    } else if (typeof module !== 'undefined' && module.exports){
        module.exports = factory();
    } else {
        global.ReconnectingWebSocket = factory();
    }
})(this, function () {

    if (!('WebSocket' in window)) {
        return;
    }

    function ReconnectingWebSocket(url, protocols, options) {

        // Default settings
        // 默认参数
        var settings = {

            /** Whether this instance should log debug messages. */
            debug: false,

            /** Whether or not the websocket should attempt to connect immediately upon instantiation. */
            automaticOpen: true,

            /** The number of milliseconds to delay before attempting to reconnect. */
            reconnectInterval: 1000,
            /** The maximum number of milliseconds to delay a reconnection attempt. */
            maxReconnectInterval: 30000,
            /** The rate of increase of the reconnect delay. Allows reconnect attempts to back off when problems persist. */
            reconnectDecay: 1.5,

            /** The maximum time in milliseconds to wait for a connection to succeed before closing and retrying. */
            timeoutInterval: 2000,

            /** The maximum number of reconnection attempts to make. Unlimited if null. */
            maxReconnectAttempts: null,

            /** The binary type, possible values 'blob' or 'arraybuffer', default 'blob'. */
            binaryType: 'blob'
        }
        if (!options) { options = {}; }

        // Overwrite and define settings with options if they exist.
        // 参数合并
        // TODO：该库把这些参数都合并到this上了，也就是合并到了原型对象上
        for (var key in settings) {
            if (typeof options[key] !== 'undefined') {
                this[key] = options[key];
            } else {
                this[key] = settings[key];
            }
        }

        // These should be treated as read-only properties

        /** The URL as resolved by the constructor. This is always an absolute URL. Read only. */
        this.url = url;

        /** The number of attempted reconnects since starting, or the last successful connection. Read only. */
        this.reconnectAttempts = 0;

        /**
         * The current state of the connection.
         * Can be one of: WebSocket.CONNECTING, WebSocket.OPEN, WebSocket.CLOSING, WebSocket.CLOSED
         * Read only.
         */
        this.readyState = WebSocket.CONNECTING;

        /**
         * A string indicating the name of the sub-protocol the server selected; this will be one of
         * the strings specified in the protocols parameter when creating the WebSocket object.
         * Read only.
         */
        this.protocol = null;

        // Private state variables

        // 缓存this
        var self = this;
        var ws;
        var forcedClose = false;
        // 是否连接超时了
        var timedOut = false;
        // 创建一个div，用于自定义事件的dom载体
        var eventTarget = document.createElement('div');

        // Wire up "on*" properties as event handlers
        // 监听open、close、message等自定义事件，
        // 响应的处理函数对应指定为实例对象上的onopen、onclose、onmessage等方法
        eventTarget.addEventListener('open',       function(event) { self.onopen(event); });
        eventTarget.addEventListener('close',      function(event) { self.onclose(event); });
        eventTarget.addEventListener('connecting', function(event) { self.onconnecting(event); });
        eventTarget.addEventListener('message',    function(event) { self.onmessage(event); });
        eventTarget.addEventListener('error',      function(event) { self.onerror(event); });

        // Expose the API required by EventTarget
        /**
         * 将实例对象上的addEventListener等监听函数绑定为eventTarget对象上的监听函数，
         * 这样用户在使用addEventListener时实际上创建的是eventTarget的监听事件，
         * 因此eventTarget.dispatchEvent分发事件时才可以触发用户设置的监听事件
         */
        this.addEventListener = eventTarget.addEventListener.bind(eventTarget);
        this.removeEventListener = eventTarget.removeEventListener.bind(eventTarget);
        this.dispatchEvent = eventTarget.dispatchEvent.bind(eventTarget);

        /**
         * This function generates an event that is compatible with standard
         * compliant browsers and IE9 - IE11
         *
         * This will prevent the error:
         * Object doesn't support this action
         *
         * http://stackoverflow.com/questions/19345392/why-arent-my-parameters-getting-passed-through-to-a-dispatched-event/19345563#19345563
         * @param s String The name that the event should use
         * @param args Object an optional object that the event will use
         */
        /**
         * 创建自定义事件，
         * TODO：这里不是要new Event的原因在于要兼容ie
         */
        function generateEvent(s, args) {
        	var evt = document.createEvent("CustomEvent");
        	evt.initCustomEvent(s, false, false, args);
        	return evt;
        };

        /**
         * 创建WS连接
         * @param {boolean} reconnectAttempt 是否尝试重连
         */
        this.open = function (reconnectAttempt) {
            ws = new WebSocket(self.url, protocols || []);
            ws.binaryType = this.binaryType;

            if (reconnectAttempt) {
                // 禁止重连次数大于用户设置的最大连接次数
                if (this.maxReconnectAttempts && this.reconnectAttempts > this.maxReconnectAttempts) {
                    return;
                }
            } else {
                // 触发eventTarget的connecting自定义事件
                // 此时用户的onconnecting钩子函数会被调用
                eventTarget.dispatchEvent(generateEvent('connecting'));
                this.reconnectAttempts = 0;
            }

            if (self.debug || ReconnectingWebSocket.debugAll) {
                console.debug('ReconnectingWebSocket', 'attempt-connect', self.url);
            }

            var localWs = ws;
            // 连接超时的处理逻辑
            var timeout = setTimeout(function() {
                if (self.debug || ReconnectingWebSocket.debugAll) {
                    console.debug('ReconnectingWebSocket', 'connection-timeout', self.url);
                }
                timedOut = true;
                localWs.close();
                timedOut = false;
            }, self.timeoutInterval);

            // ws连接成功的处理逻辑
            ws.onopen = function(event) {
                clearTimeout(timeout);
                if (self.debug || ReconnectingWebSocket.debugAll) {
                    console.debug('ReconnectingWebSocket', 'onopen', self.url);
                }
                // 连接成功后更新相关状态
                // 当前类是WebSocket的模拟实现，因此要把ws相关的属性更新到当前类上
                self.protocol = ws.protocol;
                self.readyState = WebSocket.OPEN;
                self.reconnectAttempts = 0;

                // 触发eventTarget的自定义open事件
                // 因此用户设置onopen钩子会被触发
                var e = generateEvent('open');
                e.isReconnect = reconnectAttempt;
                reconnectAttempt = false;
                eventTarget.dispatchEvent(e);
            };

            // ws连接关闭的处理逻辑
            ws.onclose = function(event) {
                clearTimeout(timeout);
                ws = null;
                // 用户手动调用ws.close()时，不会触发重连操作
                if (forcedClose) {
                    self.readyState = WebSocket.CLOSED;
                    eventTarget.dispatchEvent(generateEvent('close'));
                } else {
                    self.readyState = WebSocket.CONNECTING;
                    // 触发eventTarget的自定义事件connecting
                    // 此时用户设置的onconnecting钩子函数会被触发
                    var e = generateEvent('connecting');
                    e.code = event.code;
                    e.reason = event.reason;
                    e.wasClean = event.wasClean;
                    eventTarget.dispatchEvent(e);

                    // 当不是重连操作且没有连接超时，才触发用户设置的onclose事件
                    // TODO: 此处存疑，为什么要判断连接没有超时
                    if (!reconnectAttempt && !timedOut) {
                        if (self.debug || ReconnectingWebSocket.debugAll) {
                            console.debug('ReconnectingWebSocket', 'onclose', self.url);
                        }
                        eventTarget.dispatchEvent(generateEvent('close'));
                    }

                    // 设置重连的时间，根据速率和连接次数依次变化， 例如第四次连接: 2000 * (1.5的四次方)
                    // 连接时间不会超过设置的最大重连时间，默认最大重连时间是30秒
                    var timeout = self.reconnectInterval * Math.pow(self.reconnectDecay, self.reconnectAttempts);
                    setTimeout(function() {
                        self.reconnectAttempts++;
                        self.open(true);
                    }, timeout > self.maxReconnectInterval ? self.maxReconnectInterval : timeout);
                }
            };

            // ws收到消息的处理逻辑
            ws.onmessage = function(event) {
                if (self.debug || ReconnectingWebSocket.debugAll) {
                    console.debug('ReconnectingWebSocket', 'onmessage', self.url, event.data);
                }
                // 触发用户设置的onmessage钩子函数
                var e = generateEvent('message');
                e.data = event.data;
                eventTarget.dispatchEvent(e);
            };

            // ws出现错误的处理逻辑
            ws.onerror = function(event) {
                if (self.debug || ReconnectingWebSocket.debugAll) {
                    console.debug('ReconnectingWebSocket', 'onerror', self.url, event);
                }
                // 触发用户设置的onerror钩子函数
                eventTarget.dispatchEvent(generateEvent('error'));
            };
        }

        // Whether or not to create a websocket upon instantiation
        // 是否在实例化时就直接进行ws连接
        if (this.automaticOpen == true) {
            this.open(false);
        }

        /**
         * Transmits data to the server over the WebSocket connection.
         *
         * @param data a text string, ArrayBuffer or Blob to send to the server.
         */
        this.send = function(data) {
            if (ws) {
                if (self.debug || ReconnectingWebSocket.debugAll) {
                    console.debug('ReconnectingWebSocket', 'send', self.url, data);
                }
                return ws.send(data);
            } else {
                throw 'INVALID_STATE_ERR : Pausing to reconnect websocket';
            }
        };

        /**
         * Closes the WebSocket connection or connection attempt, if any.
         * If the connection is already CLOSED, this method does nothing.
         */
        this.close = function(code, reason) {
            // Default CLOSE_NORMAL code
            if (typeof code == 'undefined') {
                code = 1000;
            }
            // 手动关闭的ws，不会在ws.onclose时触发自动重连逻辑
            forcedClose = true;
            if (ws) {
                ws.close(code, reason);
            }
        };

        /**
         * Additional public API method to refresh the connection if still open (close, re-open).
         * For example, if the app suspects bad data / missed heart beats, it can try to refresh.
         */
        // 刷新
        // 实现逻辑为：主动关闭ws，然后会在ws.onclose时触发重新连接
        this.refresh = function() {
            if (ws) {
                ws.close();
            }
        };
    }

    /**
     * An event listener to be called when the WebSocket connection's readyState changes to OPEN;
     * this indicates that the connection is ready to send and receive data.
     */
    // 设置ws相关事件监听函数的默认值，为空函数
    ReconnectingWebSocket.prototype.onopen = function(event) {};
    /** An event listener to be called when the WebSocket connection's readyState changes to CLOSED. */
    ReconnectingWebSocket.prototype.onclose = function(event) {};
    /** An event listener to be called when a connection begins being attempted. */
    ReconnectingWebSocket.prototype.onconnecting = function(event) {};
    /** An event listener to be called when a message is received from the server. */
    ReconnectingWebSocket.prototype.onmessage = function(event) {};
    /** An event listener to be called when an error occurs. */
    ReconnectingWebSocket.prototype.onerror = function(event) {};

    /**
     * Whether all instances of ReconnectingWebSocket should log debug messages.
     * Setting this to true is the equivalent of setting all instances of ReconnectingWebSocket.debug to true.
     */
    ReconnectingWebSocket.debugAll = false;

    // 拷贝WebSocket的静态属性到ReconnectingWebSocket上
    // 因为ReconnectingWebSocket要实现和ReconnectingWebSocket相同的api
    ReconnectingWebSocket.CONNECTING = WebSocket.CONNECTING;
    ReconnectingWebSocket.OPEN = WebSocket.OPEN;
    ReconnectingWebSocket.CLOSING = WebSocket.CLOSING;
    ReconnectingWebSocket.CLOSED = WebSocket.CLOSED;

    return ReconnectingWebSocket;
});
```

该库websocket断开重连的核心逻辑如下图所示:

![image](https://note.youdao.com/yws/res/18881/065813815EE641ADA256581F2115112F)

- 首先一个立即执行函数实现各环境模块支持
- 创建`ReconnectingWebSocket`类，该类和`WebSocket`有着相同的语法实现
- 定义一些重连的基本参数，比如断开重连的时间间隔、重连次数、最大重连时间间隔等
- 创建自定义事件对象`eventTarget`，后续用户所有的`ws.onopen`、`ws.addEventListener('open', () => {}, false)`等钩子函数，都通过该事件对象触发。
- 自定义事件的实现考虑了IE的兼容
- 将`ReconnectingWebSocket`类实例上的`addEventListener`函数代理为`eventTarget`自定义事件模型上的同名函数，因此用户使用ws时设置的监听函数可以被如期触发。
- 实例化`Webscoket`：
    - 在`ws`连接成功时通过自定义事件触发`open`事件
    - 在`ws`主动关闭时，通过自定义事件触发`close`事件。
    - 在`ws`意外关闭时，进行重连操作。重连逻辑就是重新开始实例化`Websocket`进行后续的逻辑。
    - 在`ws`收到消息时通过自定义事件触发`message`事件
    - 在`ws`出错时通过自定义事件触发`error事件`
- `ReconnectingWebSocket`类实例上的`open`、`close`等方法，仅仅是对ws上的方法进行调用。
- 把`Websocket`上的一些静态属性赋值给`ReconnectingWebSocket`

## 总结

该库主要看Websocket断开重连的思想以及自定义事件的实现。