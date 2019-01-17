<template>
  <div class="content-wrap">
    <div style="margin-bottom: 15px;">
      账号名：
      <input type="text" v-model="userId" placeholder="账号名"/>
      登录密码:
      <input type="password" v-model="password" placeholder="登录密码"/>
      <button v-on:click="connect">登录</button>
    </div>
    <div class="chat-box-wrap">
      <!--
      <div class="left-msg-wrap">
        <span><span class="nickname">小明:</span><span>你好呀！</span></span>
      </div>
      <div class="right-msg-wrap">
        <span><span>我很好，你怎么样?</span><span class="nickname">:孙尚香</span></span>
      </div>
      -->
      <div v-for="item in messageArr" >
        <div v-if="item.userId==userId"  class="left-msg-wrap">
          <span><span class="nickname"><span>{{item.userId}}</span>:</span><span>{{item.content}}</span></span>
        </div>
        <div v-else class="right-msg-wrap">
          <span><span class="nickname"><span>{{item.userId}}</span>:</span><span>{{item.content}}</span></span>
        </div>
      </div>
    </div>
    <div class="msg-send-box">
      <div style="margin-bottom:15px;">
        <span>
          好友列表:
        </span>
          <span>
          <select v-model="toUserId">
            <option v-for="item in users" v-bind:value="item.account">{{item.account}}</option>
          </select>
        </span>
      </div>

      <span>
        <input type="text" v-model="messageContent" placeholder="请输入聊天消息!"/>
        <button v-on:click="sendMessage">发送</button>
      </span>
    </div>
    <div style="margin-top: 15px;"><span>{{connectInfo}}</span></div>
  </div>
</template>

<script>
const axios = require('axios')
export default {
  name: 'HelloWorld',
  data () {
    return {
      userId: '',
      password: '',
      toUserId: '2',
      messageContent: '',
      ws: undefined,
      messageArr: [],
      connectInfo: '',
      users: {}
    }
  },
  created () {
    var self = this
    axios
      .get('http://localhost:3000/user/getUsers')
      .then(response => {
        var len = response.data.body.length
        self.toUserId = response.data.body[0].account
        for (var i = 0; i < len; i++) {
          var user = response.data.body[i]
          self.users[user._id] = user
          self.$forceUpdate() // 强制刷新，解决页面不会重新渲染的问题
        }
      })
      .catch(error => {
        console.log(error)
        this.errored = true
      })
  },
  methods: {
    connect () {
      var self = this
      // 1、登录用户账号
      axios
        .post('http://localhost:3000/user/login', {
          account: self.userId,
          password: self.password
        })
        .then(response => {
          if (response.data.code == 0) {
            console.log('用户' + self.userId + '登录成功!')
            // 2、登录成功，调用连接websocket服务器的方法
            connectWsServer()
          } else {
            console.log('用户' + self.userId + '登录失败!')
          }
        })
        .catch(error => {
          console.log(error)
          this.errored = true
        })
      /**
       * 声明连接推送的方法，里面是对具体操作步骤的封装
       */
      function connectWsServer () {
        // 服务器地址：端口号?参数的配置
        var url = 'ws://192.168.43.220:3001/auth?account=' + self.userId
        // 创建客户端websocket并想服务器发起连接请求
        var ws = new WebSocket(url)
        self.ws = ws // 将客户端websocket暴露出去，方便其他地方使用
        /**
         *  3、监听连接成功事件，服务器认证成功并连接成功后会回调此方法
         */
        ws.onopen = function () {
          console.log('open')
          self.connectInfo = self.userId + '连接聊天服务成功,url：' + url
        }
        /**
         * 4、监听服务器发送回来的消息
         * @param evt
         */
        ws.onmessage = function (evt) {
          console.log(evt.data)
          var msg = JSON.parse(evt.data)// 将接收到的json字符串消息转换成js对象，便于处理
          if (msg.type == 'chatMsg') { // 如果收到的是聊天消息
            self.messageArr.push(msg) // 将消息渲染到聊天窗口上
          } else if (msg.type == 'notifyMsg') { // 如果接收到的是通知消息
            if (msg.code == 'userNotOnline') { // 如果通知消息是提示消息接收者不在线，则提示用户
              self.connectInfo = msg.msg
            }
          }
        }
        /**
         * 监听连接关闭事件
         * 情景跟服务器关闭事件类似
         * @param evt
         */
        ws.onclose = function (evt) {
          console.log('WebSocketClosed!')
        }
        /**
         * 监听错误事件
         * @param evt
         */
        ws.onerror = function (evt) {
          console.log('WebSocketError!')
          self.connectInfo = self.userId + '连接聊天服务失败,url：' + url
        }
      }
    },
    /**
     * 发送聊天消息到服务器端
     */
    sendMessage () {
      var msg = {
        type: 'chatMsg', // 消息类型为聊天消息
        userId: this.userId,// 自己的用户Id
        toUserId: this.toUserId,// 消息接收者的用户Id
        nickname: this.users[this.userId],// 消息发送者的昵称
        content: this.messageContent// 消息内容
      }
      console.log('send msg:' + JSON.stringify(msg))
      this.ws.send(JSON.stringify(msg))// 将消息对象转换为json字符串后再通过客户端websocket发送给服务器的websocket进行接收
      this.messageArr.push(msg) // 将发送自己发送的消息渲染到聊天窗口里
      this.messageContent = '' // 将消息输入框清空
      this.connectInfo = '消息发送成功！' // 提示消息发送成功
    }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
  .content-wrap{
    width:80%;
    margin:0 auto;
  }
  .chat-box-wrap{
    height: 60vh;
    width:100%;
    overflow-y: auto;
    border: 1px solid #ccc;
    background-color: #c3c3c3;
    padding-top:30px;
    padding-left: 15px;
    padding-right:15px;
    margin-bottom: 20px;
  }
  .left-msg-wrap{
    text-align: left;
  }
  .nickname{
    margin-right:15px;
  }
  .right-msg-wrap{
    text-align: right;
  }

</style>
