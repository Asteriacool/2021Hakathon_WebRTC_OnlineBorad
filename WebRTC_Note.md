### 2021 Spring Amazon Hackathon

#### 一、WebRTC

WebRTC提供的三个接口

##### 1.getUserdia

请求获取用户的媒体流

* 视频流video:ture
  * 获取video对象后，给video对象添加入流video.srcObject = localStream
* 音频流audio:true
* 使用时要注意该API的兼容性问题

##### 2.RTCPeerConnection

建立本地用户和远端用户之间的连接。（第一次连接需要借助服务器来连接，需要服务器来进行中转，当第一次连接上之后就不需要通过服务器来完成）

* 创建RTCPeerConnection实例

```javascript
let PeerConnection = window.RTCPeerConnection || window.mozRTCPeeerConnection || window.webkitRTCPeerConnection

let peer = new PeerConnection(iceServers)
```

iceServer参数中包括两个属性（用于NAT穿透

）

* stun
* turn

```javascript
iceServers: [
	{url:"stun:stun.l.google.com:19302"} //谷歌的公众服务
	{url:"turn:***", 
  			username:***,//用户名 
 				credential:***//密码}
]
```

技术解释：

(1)NAT：网络地址转换——实现不同局域网之间的通信

* 实现方式：使用STUN和TURN来进行NAT穿透。该过程是通过STUN Server来进行NAT穿透，如果无法穿透则需要使用TURN Serve来进行中转。

(2)SDP：会话描述协议，可以完成多种类型的传输。对于本地来说，是本地媒体元数据。

(3)ICE：交互式连接建立。用于在offer/answer模式下的NAT传输协议，主要用于UDP下多媒体会话的建立，使用了STUN协议以及TURN协议。

* 交换本地(localPeer)和远端用户(remotePeer)的数据描述，使用offer和answer来进行NAT穿透，连接端与端
  * 生成本地sdp数据描述（offer类型）：localPeer调用createOffer()API来创建一个offer类型的sdp（业务数据），并使用setLocalDescriptiob()将其添加到localDescription
  * 远端接受本地用户的数据描述：remotePeer接收到localPeer的localDescription，并使用setRemoteDescription将其添加到自己的RemoteDescription
  * 生产远端SDP数据描述（answer类型）：remotePeer通过createAnswer()来创建一个answer类型的sdp，并将其添加到自己的LocalDescription
  * 本地p2p连接建立完成（如果是网络的p2p，还需要socket.io来进行数据传输）

```
localPeer.createOffer()
.then(offer => localPeer.setLocalDescription(offer))
.then(() => remotePeer.setRemoteDescription(localPeer.localDescription))
.then(() => remotePeer.createAnswer)
.then(answer => remotePeer.setLocalDescription(answer))
.then(() => localPeer.setRemoteDescription(remotePeer.localDescription))
```

* 交换ICE网络信息，用于联网的时候进行网络信息交换
  * (1)进行两端ICE协议连接：
    * RTCPeerConnection.onIceCandidate():该API用于监视本地ice网络的变化，如果有变化，就将其使用socket.io发送出去
    * RTCPeerConnection.addIceCandidate():该API用于将受到的ice添加到本地的RTCPeerConnection实例中
  * (2)使用RTCPeerConnection实例中的方法对媒体流进行操作，包括：
    * addstream()添加本地的媒体流
    * onaddstream()检测本地的媒体流
    * 注意onaddstream()在接受端answer的setRemoteDescription执行完成后会立即执行，也就是我们不能再p2p创建完成后再使用addstream来添加流

##### 3.RTCDataChannel

用于p2p连接（在第二步已经完成）的用于通道

* 用createDataChannel()创建通信的通道实例channel
* 通过channel.send()的方法，channel主动向已经建立连接的通道发送数据
* 使用peer.ondatachannel()来监听channel的状态
  * channel.onopen：通道打开
  * channel.onclose：通道关闭
  * channel.onmessage：通道收到的信息

```javascript
//发送数据hello
channel.send(JSON.stringify('hello'))

//监听channel的状态
peer.ondatachannel = (event) =>{
	var channel = event.chanel
	channel.binaryType = 'arraybuffer'
	
	channel.onopen = (event) => {
		//连接成功
		console.log('channel onopen')
	}
	
	chnnel.onclose = function(event){
		//连接关闭
		console.log('channel onclose')
	}
	
	channel.onmessage = (event) => {
		//收到消息
		let data = JSON.parse(event.data)
		console.log('channel onmessage',data)
	}
}
```



#### 二、Websocket介绍

##### 1.数据传输的几种方式

###### TCP传输协议

* 面向有连接的通信服务，只有在确认对端存在式才会收发数据
* 控制通信流量浪费
* 如果丢包，则进行重发控制
* 如果前面的数据包不到达，后面的数据包会继续等待

###### UDP传输

* 简单的、面向无连接的协议，提供的不一定是可靠的传输服务
* 不管双方的状态都直接传输
* 不能保证数据传输的可靠性，但是传输效率很高
* 类似手机发短信

###### Webstocket协议传输

* 在该传输方式出现之前，主要靠Ajax轮询的方式，缺点是容易给服务器造成压力
  * 客户端按照规定的时间定时向服务器端发出Ajax请求，服务器收到请求之后马上返回响应信息并关闭连接
  * 需要服务器有很快的处理速度并关闭连接
* HTML5开始提供的一种在单个TCP连接上进行全双工通讯的协议
  * 全双工通信：通信的双方可以同时发送和接受信息的交互方式
* 特点：简化数据交换，允许服务器主动向客户端推送数据。在WebSocketAPI中，浏览器和服务器只需要完成一次握手，两者之间就可以建立持久性连接，并进行双向数据传输

##### 2.传统HTTP与Websocket的异同

##### 不同点

1. HTTP是单向数据流，客户端向服务端发送请求，服务端响应并返回数据；WebSocket连接后可以实现客户端和服务端双向数据传递
2. 由于是新的协议，HTTP的url使用"http//"或"https//"开头；Websocket的url使用"ws//"开头
3. 在html5之前，因为http协议是无状态的，要实现浏览器与服务器的实时通讯，如果不使用flash、applet等浏览器插件的话，就需要定期轮询服务器来获取信息。这就造成了一定的延迟和大量的网络通讯，但是Websocket是全双工式的通信方式

##### 相同点

1. 都需要建立TCP连接
2. 都是属于七层协议中的应用层协议

##### 3.Websocket的工作机制

* 浏览器通过JavaScript向服务器发出建立websocket连接的HTTP请求
  * 这个请求和通常的HTTP请求不同，包含了一些附加头信息，其中附加头信息"Upgrade:WebSocket"表明这是一个申请协议升级的HTTP请求。
* 服务器端解析这些附加的头信息然后产生应答信息返回给客户端
* 客户端和服务器端的Websocket连接完成建立
* 连接建立之后，客户端和服务器端就可以通过TCP连接直接交换数据
  * 因为Websocket连接本质上就是一个TCP连接，所以在数据传输的稳定性和数据传输量的大小方面，和传统轮询技术比较，具有很大的性能优势。并且这个连接会持续存在直到客户端或者服务器端的某一方主动的关闭连接。
* ※WebSocket是可以和HTTP共用监听端口的，也就是他可以使用公用端口完成Soket任务

#### 三、Socket.io

##### 1.组成

Socket.io是利用Websocket进行双向数据通信的实现方式

* 它并不是Websocket，它只是将Websocket和轮询Polling机制以及其他的实时通信方式封装成了通用的接口，并且在服务器端实现了这些实时机制响应的代码
* Websocket仅仅是Socket.io实现实时通信的一个子集
  * 因此Websocket客户端连接不上Socket.io服务端，当然Socket.io客户端也连接不上Websocket服务端
  * Adobe® Flash® Socket
  * AJAX long polling
  * AJAX multipart streaming
  * Forever Iframe
  * JSONP Polling
* 需要使用到的Socket.io的API
  * socket.on('event',() => {})监听socket触发的事件
  * socket.emit('event',() =>{})主动发送
  * socket.join('room',()=>{})加入房间
  * socket.leave('room',()=>{})离开房间
  * socket.to(room | socket.id) | socket.in(room | socket.id) 指定房间或者服务器

##### 2.利用socket.io实现通信的过程

(1)建立客户端和服务器端（端口号为3000）的相互连接

```javascript
// html
// 引入 <script src="https://cdn.bootcss.com/socket.io/2.2.0/socket.io.js"></script>
// 连接3000端口
var socket = io('ws://localhost:3000/‘)


// server.js 用来部署服务器
// 监听连接
// io是服务器端的，socket是客户端的
io.on('connection', socket => { ... })


// 监听关闭
io.on('disconnect', socket => {})
```

(2)通过Socket.io来实现WebRTC的第一次连接

```JavaScript
// A 向 B 的p2p
// html


// A
// user 是全局变量，存在sessionStorage中， 创建时候获取
var user = window.sessionStorage.user || ''

// 发给服务器改socket的名称A
socket.emit('createUser', 'A')

// 处理兼容性
let PeerConnection = window.RTCPeerConnection || window.mozRTCPeerConnection || window.webkitRTCPeerConnection
var peer = new PeerConnection()

// 创建A端的offer
peer.createOffer()
    .then(offer => {
        // 设置A端的本地描述
        peer.setLocalDescription(offer, () => {
       
        // socket发送offer和房间
        socket.emit('offer', {offer: offer, user: 'B'})
        })
    })

// 监听本地的ice变化，有则发送个B
peer.onicecandidate = (event) => {
    if (event.candidate) {
    ![](https://user-gold-cdn.xitu.io/2019/6/3/16b1b606f637e98e?w=1829&h=1005&f=gif&s=4145004)

        // B  
        // user 是全局变量，存在sessionStorage中， 创建时候获取
        var user = window.sessionStorage.user || ''

        // 发给服务器socket的名称
        socket.emit('createUser', 'A')
        let PeerConnection = window.RTCPeerConnection || window.mozRTCPeerConnection || window.webkitRTCPeerConnection
        
        var peer = new PeerConnection()

        // 接受服务器端发过来的offer辨识的数据
        socket.on('offer', date => {
        
            // 设置B端的远程offer 描述
            peer.setRemoteDescription(data.offer, () => {
        
            // 创建B的Answer
            peer.createAnswer()
                .then(answer => {
                // 设置B端的本地描述
                peer.setLocalDescription(answer, () => {
                    socket.emit('answer', {answer: answer, user: 'A'})
                 })
            })
        })
    })
    socket.on('ice', data => {
        // 设置B ICE
        peer.addIceCandidate(data.candidate);
    })

    socket.emit('createUser', 'B')


    // server.js
    // 用于接受客户端的用户名对应的服务器
    const sockets = {}
    
    // 保存user
    const users = {}
    io.on('connection', data => {
        // 创建账户
        socket.on('createUser', data => {
        let user = new User(data)
        users[data] = user
        sockets[data] = socket})

    socket.on('offer', data => {
        // 通过B的socket的id只发送给B
        socket.to(sockets[data.user].id).emit('offer', data)})
    
    socket.on('answer', data => {
        // 通过B的socket的id只发送给A
        socket.to(sockets[data.user].id).emit('answer', data)})
    
        socket.on('ice', data => {
            // ice发送给B
        socket.to(sockets[data.user].id).emit('ice', data)})
})

```

这里server.js的作用是进行中转

socket广播的其他API

* io.emit()对连接了服务器的所有客户端进行广播，比如显示房间信息
* io.to(room).emit()对一个房间中所有客户端进行广播，用于房间内的通知
* socket.to(room).emit()发送给房间中除了自己以外的服务器
* socket.emit()发送给服务器自己
* socket.ti(socket.id).emit()发送给指定的服务器

##### 3.一对多和多对多通信的建立

因为是p2p，所以要实现多对多，就可以变成每个的一对一。就是用过每一个端都进行p2p连接，这里我们需要注意添加的顺序问题，当有人进入房间时，进入的人和房间的人每一个进行P2P，但是已经进入的人就只需要和新进入的人进行p2p。

在每一个客户端都使用了一个数组进行存储，加入的和现有的user进行不同的标识。

#### 四、实时画板的实现

实现实时画板的几种方法：

* 通过socket.io来进行主动的数据传输
* 通过将canvas变成数据流，并且通过addSream和onAddSteam来进行，将流传输并用video进行接收流
* 通过RTCDataChannel来实现
  * 主动发送数据到其他端，其他端在自己的canvas上进行绘画

1.在HTML中定义canvas标签

```HTML
<canvas id="canvas" class="cursor1" width="500" height="500"></canvas>
<script src="canvas-demo.js"></script>
```

2.书写布局

SVG标签允许定义需要重复使用的图形元素。建议把所有需要再次使用的元素定义在defs元素里面，以增加SVG内容的易读性和可访问性。在defs元素中定义的标签不会直接呈现，而可以在视口的任意地方利用<use>元素呈现这些元素。

```html
<div id="actions" class="actions">
        <svg id="brush" class="icon active">
            <use xlink:href="#icon-pencil"></use>
        </svg>
        <svg id="eraser" class="icon">
            <use xlink:href="#icon-xiangpica2"></use>
        </svg>
        <svg id="save" class="icon">
            <use xlink:href="#icon-xiazai"></use>
        </svg>
        <svg id="clear" class="icon">
            <use xlink:href="#icon-delete"></use>
        </svg>
</div>

<ol class="colors">
        <li id="black" class="black"></li>
        <li id="red" class="red"></li>
        <li id="orange" class="orange"></li>
        <li id="green" class="green"></li>
        <li id="blue" class="blue"></li>
</ol>
 <script src="canvas-demo.js"></script>
```

3.设置样式

```HTML
 <link rel="stylesheet" href="style.css">
```

```css
*{margin:0;padding: 0;}
ul,ol{list-style: none;}
.icon {
            width: 1em; height: 1em;
            vertical-align: -0.15em;
            fill: currentColor;
            overflow: hidden;
        }


#canvas{
            position: fixed;
            top: 0;
            left: 0;
            background: #C5C5C5;
            display: block;
            overflow: hidden;
}
        .actions{
            position: fixed;
            top: 0;
            left: 0;
        }
        .actions> svg{
            width: 1.5rem;
            height: 1.5rem;
            margin: 0.5rem 1rem;
            transition: all 0.3s;
        }
        .actions svg.active{
            fill: orangered;
            transform: scale(1.4);
        }
        .colors{
            position: fixed;
            top: 4rem;
            left: 0.5rem;
        }
        .colors>li{
            margin: 1rem 0;
            width: 2rem;
            height: 2rem;
            border-radius: 1rem;
            box-shadow: 0px 0px 3px black;


        }
        .black{background: black}
        .red{background: red}
        .orange{background: orange}
        .green{background: green}
        .blue{background: blueviolet}
        .cursor1{cursor: url('ico/black.ico') 8 20,auto;}
        .cursor2{cursor: url('ico/red.ico') 8 20,auto;}
        .cursor3{cursor: url('ico/orange.ico') 8 20,auto;}
        .cursor4{cursor: url('ico/green.ico') 8 20,auto;}
        .cursor5{cursor: url('ico/blue.ico') 8 20,auto;}
        .xiangpica{cursor: url('ico/xiangpica.ico') 8 20,auto;}
```

4.书写canva-demo.js

##### (1)基础设置

取HTML中ID为canvas的canvas标签，并且声明canvas的内容为2d

```javascript
let canvas = document.getElementById("canvas");
let ctx = canvas.getContext('2d');
```

##### (2)设置画布的大小

```javascript
// 定义画布的宽高：将canvas与屏幕宽高设置为一致
wh()
//---------工具函数
function wh(){
    let pageWidth = document.documentElement.clientWidth;
    let pageHieght = document.documentElement.clientHeight;
    canvas.width = pageWidth;
    canvas.height = pageHieght;
}
```

##### (3)特性检测

```javascript
// 特性检测
// 通过检测是否有document.body.ontouchstart来判断是否支持触屏设备
if(document.body.ontouchstart !== undefined) {
    // 触屏设备
}else{
    // 非触屏设备
}
```

##### (4)画图事件

点击鼠标的时候监听鼠标事件，获取x轴和y轴对应视口的坐标点

- 点击鼠标——记录第一个坐标点x1(startpoint)
- 滑动鼠标——记录开始滑动的第一个坐标点x2(newpoint)，利用canvas中的stroke()事件将上述两个点连成线。将x2的坐标点赋值给x2，继续滑动鼠标，获取此时的x2，连接此时的x1和x2，以此类推
- 松开鼠标——将开关设置为false，循环终止，停止赋值

两点之间连线的工具函数

```javascript
// 两点之间连线的工具函数
function drawLine(xStart,yStart,xEnd,yEnd){
    // 开始绘制路径
    ctx.beginPath();
    // 线宽
    ctx.lineWidth = 2;
    // 起始位置
    ctx.moveTo(xStart,yStart);
    // 停止位置
    ctx.lineTo(xEnd, yEnd);
    // 描绘线路
    ctx.stroke();
    // 结束绘制
    ctx.closePath();
}
```

定义开关变量和第一个点的坐标和橡皮擦开关

```javascript
//画板控制开关
let painting = false;
//第一个点坐标
let startPoint = {x: undefined, y: undefined};
// 橡皮擦开关
let EraserEnabled = false;
```

书写画线的过程

```javascript
    // 非触屏设备
    // 鼠标点击事件(onmousedown)
    canvas.onmousedown = function(e){
        let x = e.offsetX;
        let y = e.offsetY;
        painting = true;
        if (EraserEnabled) {
            // 有橡皮擦的操作
        }
        startPoint = {x : y,y : y};
    };

    // 鼠标滑动事件(onmousemove)
    canvas.onmousemove = function(e){
        let x = e.offsetX;
        let y = e.offsetY;
        let newPoint = {x : x,y: y};
        if(painting){
            if(EraserEnabled){
                // 有橡皮擦的操作
            }else{
                drawLine(startPoint.x,startPoint.y,newPoint.x,newPoint.y);
            }
            startPoint = newPoint;
        };
    };
    // 鼠标松开事件(onmouseup)
    canvas.onmouseup = function(){
        painting = flase;
    };
```

注意，对于移动设备来说，由于设备是支持多点触摸的，因此这里的x轴和y轴需要从e.touches[0]数组中获取

```javascript
 // 触屏设备
    // 初始接触事件
    canvas.ontouchstart = function(e){
        let x = e.touches[0].clientX;
        let y = e.touches[0].clientY;
        painting = true;
        if(EraserEnabled){
            //橡皮擦操作
        }
        startPoint = {x:x,y:y};
    }
    // 接触移动事件
    canvas.ontouchmove = function(e){
        let x = e.touches[0].clientX;
        let y = e.touches[0].clientY;
        let newPoint = {x:x, y:y};
        if(painting){
            if(EraserEnabled){
                // 橡皮擦操作
            } else{
                drawLine(startPoint.x,startPoint.y,newPoint.x.newPoint.y);
            }
            startPoint = newPoint;
        };
    };
    // 接触停止事件
    canvas.ontouchend = function(){
        painting = false;
    };
```

##### (5)橡皮擦功能

- 通过EraserEnabled是否等于true开启橡皮擦功能
- 利用clearRect绘制一个空白的矩形，进行与上述划线步骤相似的操作：点击（获取初始坐标）、滑动（获取新的坐标，连线，更新坐标）、松开即可达到擦除的效果

```javascript
if(EraserEnabled){
            //橡皮擦操作
            ctx.clearRect(x-15, y-15, 30,30);
        }
```

