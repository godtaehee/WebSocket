# 웹 소켓

노드 생태계에서 웹 소켓이라는 말을 들으면 Socket.IO를 먼저 떠올리는 경우가 많다. 하지만 Socket.IO는 웹 소캣을 활용한 라이브러리일뿐 웹 소캣 그 자체는 아니다. 나중에 Socket.IO를 사용하기 위해서는 기반 기술인 웹 소켓에 대해 먼저 알아야 합니다.


## 웹 소켓이란?

웹 소켓은 HTML5에 새로 추가된 스펙으로 실시간 양방향 데이터 전송을 위한 기술이며, HTTP와 다르게 WS라는 프로토콜을 사용한다.

따라서 브라우저와 서버가 WS 프로토콜을 지원하면 사용할 수 있습니다. 최신 브라우저는 대부분 웹 소켓을 지원하고, 노드에서는 ws나 Socket.IO 같은 패키지를 통해 웹 소켓을 사용할 수 있다.

## 웹 소켓이 나오기 이전 (Polling)

웹 소켓이 나오기 이전에는 HTTP 기술을 사용하여 실시간 데이터 전송을 구현했다. 그중 한 가지가 폴링(polling)이라고 불리는 방식이다.

HTTP가 클라이언트에서 서버로 향하는 단방향 통신이므로 주기적으로 서버에 새로운 업데이트가 있는지 확인하는 요청을 보낸 후, 있다면 새로운 내용을 가져오는 단순 무식한 방법이었습니다.

그러다가 HTML5가 나오면서 웹 브라우저와 웹 서버가 지속적으로 연결된 라인을 통해 실시간으로 데이터를 주고받을 수 있는 웹 소켓이 등장 했습니다.

처음에 웹 소캣 연결이 이루어지고 나면 그다음부터는 계속 연결된 상태로 있으므로 따로 업데이트가 있는지 요청을 보낼 필요가 없습니다. 업데이트할 내용이 생겼다면 서버에서 바로 클라이언트에 알립니다. HTTP 프로토콜과 포트를 공유할 수 있으므로 다른 포트에 연결할 필요도 없습니다. 폴링 방식에 비해 성능도 매우 개선되었습니다.

참고로 서버센트 이벤트(Server Sent Events)(이하 SSE)라는 기술도 등장했습니다. EventSource라는 객체를 사용하는데, 처음에 한 번만 연결하면 서버가 클라이언트에 지속적으로 데이터를 보냅니다. 웹 소켓과 다른 점은 클라이언트에서 서버로는 데이터를 보낼 수 없다는 것입니다. 즉, 서버에서 클라이언트로 데이터를 보내는 단방향 통신입니다. 따라서 웹 소켓만이 진정한 양방향 통신입니다. 양방향 통신이므로 SSE에서 할 수 있는 것은 웹 소켓으로도 모두 할 수 있습니다. 하지만 주식 차트 업데이트나 SNS에서 새로운 게시물 가져오기 등 굳이 양방향 통신이 필요 없는 경우도 많습니다. 서버에서 일방적으로 데이터를 내려주기만 하면 되기 때문이죠. 다음 장에서 경매 시스템을 만들 때 SSE 기술을 사용해볼 것입니다.

`Socket.IO`는 웹 소켓을 편리하게 사용할 수 있도록 도와주는 라이브러리입니다. Socket.IO는 웹 소켓을 지원하지 않는 IE9과 같은 브라우저에서는 알아서 웹 소켓 대신 폴링 방식을 사용하여 실시간 데이터 전송을 가능하게 합니다. 클라이언트 측에서 웹 소켓 연결이 끊겼다면 자동으로 재연결을 시도하고, 채팅방을 쉽게 구현할 수 있도록 메서드를 준비해 두었습니다.


## ws Module

```javascript
import WebSocket from "ws";

module.exports = (server) => {
  const wss = new WebSocket.Server({ server });

  wss.on("connection", (ws, req) => {
    // 웹소켓 연결 시
    const ip = req.headers["x-forwarded-for"] || req.connection.remoteAddress;
    console.log("새로운 클라이언트 접속", ip);
    ws.on("message", (message) => {
      // 클라이언트로부터 메시지
      console.log(message);
    });
    ws.on("error", (error) => {
      // 에러 시
      console.error(error);
    });
    ws.on("close", () => {
      // 연결 종료 시
      console.log("클라이언트 접속 해제", ip);
      clearInterval(ws.interval);
    });

    ws.interval = setInterval(() => {
      // 3초마다 클라이언트로 메시지 전송
      if (ws.readyState === ws.OPEN) {
        ws.send("서버에서 클라이언트로 메시지를 보냅니다.");
      }
    }, 3000);
  });
};
```

ws 모듈을 불러온 후 익스프레스 서버를 웹 소켓 서버와 연결했다. 익스프레스(HTTP)와 웹 소켓(WS)은 같은 포트를 공유할 수 있으므로 별도의 작업이 필요하지 않다.

![Screen Shot 2021-10-03 at 11 05 59 AM](https://user-images.githubusercontent.com/44861205/135736911-98034cf5-6d19-4f4e-8265-833c2776530f.png)

연결 후에는 웹 소켓 서버(wss)에 이벤트 리스너를 붙입니다. 웹 소켓은 이벤트 기반으로 작동한다고 생각하면 됩니다. 실시간으로 데이터를 전달받으므로 항상 대기하고 있어야 합니다. `connection` 이벤트는 클라이언트가 서버와 웹 소켓 연결을 맺을 때 발생합니다. `req.headers['x-forwarded-for']`는 프록시 서버의 IP주소를 알고싶을때 사용하며 `req.connection.remoteAddress`는 클라이언트의 IP주소를 알아내기위한 유명한 방법입니다. 하지만 노드 13이상에서부터는 deprecated된 방법이므로 `req.socket.remoteAddress`를 사용합니다. Express의 Request객체로 `req.ip`로도 알수 있습니다. 또한 Express에서는 `proxy-addr`패키지를 사용해서도 IP를 알수 있습니다. 로컬 호스트로 접속한 경우, 크롬에서는 IP가 `::1`로 뜹니다 이 주소체계는 `IPv6`의 locallhost입니다. 다른 브라우저에서는 ::1 외에 다른 IP가 뜰 수 있습니다.

웹 소켓에서는 CONNECTING(연결 중), OPEN(열림), CLOSING(닫는 중), CLOSED(닫힘)의 4가지 상태가 있습니다. OPEN 일때만 에러 없이 메시지를 보낼 수 있습니다.

브라우저 개수만큼 서버에는 배수로 Client로부터의 메시지를 받을 것이며 브라우저가 닫히면 연결이 해제됬다는 로그가 서버측에 출력이 됩니다.

## Socket.io

```javascript
  const io = SocketIO(server, { path: "/socket.io" });
```

SocketIO 객체의 두 번째 인수로 옵션 객체를 넣어 서버에 관한 여러가지 설정을 할 수 있습니다. path는 클라이언트가 접속할 경로를 나타내며 클라이언트도 이 경로와 일치하는 path를 넣어야합니다.

connection 이벤트는 클라이언트가 접속했을 때 발생하고, 콜백으로 소켓 객체(socket)을 제공합니다.

socket.request속성으로 요청 객체에 접근할 수 있습니다.

socket.request.res로는 응답 객체에 접근할 수 있습니다.

socket.id로 소켓 고유의 아이디를 가져올 수 있습니다. 이 아이디로 소켓의 주인이 누구인지 특정할 수 있습니다.

```javascript
socket.emit("news", "Hello Socket.IO");
```

socket에도 이벤트 리스너를 붙일수있으며 disconnect news라는 이벤트를 만들었습니다. 이 news이벤트는 사용자가 마음대로 정의한 이벤트 이름이여 이것이 `ws`모듈과 `socket.io`의 다른 점입니다.

클라이언트에서 news이벤트를 받기위해서는 똑같이 클라이언트에도 new 이벤트 리스너를 만들어두어야 합니다.

```javascript
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>GIF 채팅방</title>
</head>
<body>
  <div>F12를 눌러 console 탭과 network 탭을 확인하세요.</div>
  <script src="/socket.io/socket.io.js"></script>
  <script>
    // http를 사용함
    const socket = io.connect('http://localhost:3000', {
      path: '/socket.io',
      transports: ['websocket'],
    });
    socket.on('news', function (data) {
      console.log(data);
      socket.emit('reply', 'Hello Node.JS');
    });
  </script>
</body>
</html>
```

`/socket.io/socket.io.js`는 Socket.IO에서 클라이언트로 제공하는 스크립트이며, 실제 파일이 아닙니다. 이 스크립트를 통해 서버와 유사한 API로 웹 소켓 통신이 가능합니다. 스크립트가 제공하는 io객체에 서버 주소를 적어 연결합니다. 이때 http를 사용하였는데 Socket.IO는 처음에는 폴링 방식으로 서버와 연결을 하기 때문입니다. 폴링 연결(xhr) 후, 웹 소켓을 사용할 수 있다면 웹 소켓으로 업그레이드를 합니다.

웹 소켓을 지원하지 않는 브라우저는 폴링 방식으로, 웹 소켓을 지원하는 브라우저는 웹 소켓 방식으로 사용 가능한 것입니다.

처음 부터 웹 소켓만 사용하고 싶다면, 클라이언트에서 다음과 같이 `transport: ['websocket']`옵션을 주면됩니다.


