<template>
  <div className="hello">
    <div id="inspectionCountdown" className="font-time">00:00:00</div>
  </div>
</template>

<script>
import {ref} from 'vue'
import Stomp from "stompjs";
import $ from 'jquery'

export default {
  name: 'HelloWorld',
  props: {
    msg: String
  },

  setup() {
    const count = ref(1);
    let lockReconnect = false;
    let stompClient = null;
    // 机组编码， 实际环境中用选择的单元替换
    let crewCode = "01";
    let seconds = 0;
    let nextInterId = 0;
    // 查询倒计时接口
    // 倒计时方法
    function countdown() {
      clearInterval(nextInterId);
      nextInterId = setInterval(flashText, 1000);
    }

    // 获取倒计时数据接口， 参考 demo
    async function getCountdownData() {
      try {
        $.ajax({
          url: "http://127.0.0.1:8705/job/crew",
          type: "GET",
          data: {
            crewCode: crewCode
          },
          success: resp => {
            console.log(resp)
            seconds = Number(resp.content);
            // 执行倒计时方法
            countdown();
          }
        })
      } catch (error) {
        console.log("请求失败")
      }
    }

    // 将 秒转 换为 hh:mm:ss 格式
    function countdownTimes(secondData) {
      if (secondData <= 0) secondData = 0;
      let secondTemp = secondData % 60;
      let minute = parseInt(secondData / 60) % 60;
      let hour = parseInt(secondData / 3600) % 60;
      if (secondTemp < 10) {
        secondTemp = "0" + secondTemp;
      }
      if (minute < 10) {
        minute = "0" + minute;
      }
      if (hour < 10) {
        hour = "0" + hour;
      }
      return hour + ":" + minute + ":" + secondTemp;
    }

    // 刷新文本方法
    function flashText() {
      // 查询倒计时
      if (seconds <= 0) {
        clearInterval(nextInterId);
        return (seconds = 0);
      }
      seconds = seconds - 1;
      let oElem = document.getElementById("inspectionCountdown");
      oElem.innerText = countdownTimes(seconds);
    }

    // WebSocket 开始
    function connection() {
      const headers = {
        login: "guest",
        passcode: "guest",
        //虚拟主机，默认“/”
        host: "/demo",
        "heart-beat": "0,0",
      };
      // 建立连接对象
      const ws = new WebSocket("ws://127.0.0.1:8705/ws"); // 连接服务端提供的通信接口，连接以后才可以订阅广播消息和个人消息
      // 获取STOMP子协议的客户端对象
      stompClient = Stomp.over(ws);
      stompClient.debug = null;
      // const userCode = this.$store.getters.userInfo.userCode;
      // 向服务器发起websocket连接
      stompClient.heartbeat.outgoing = 2000;
      stompClient.heartbeat.incoming = 2000;
      stompClient.connect(
          headers,
          () => {
            stompClient.subscribe(
                `/exchange/countdown/` + crewCode,
                (msg) => {
                  console.log("消息来了", msg)
                  // 订阅服务端提供的某个topic;
                  const res = JSON.parse(msg.body);
                  switch (res.mark) {
                    case "/test":
                      // 倒计时获取
                      console.log("content", res.content)
                      seconds = Number(res.content);
                      // 执行倒计时方法
                      countdown();
                      break;
                    default:
                      break;
                  }
                },
                {
                  //绑定前端监听队列，后面的随机数保证每个客户端都有一个自己的队列
                  //队列名称
                  "x-queue-name":
                      "vue_countdown" +
                      "_" +
                      crewCode +
                      "_" +
                      Math.floor(Math.random() * (100 - 1)) +
                      1,
                  durable: true, //自动删除（客户端关闭队列自动删除，刷新页面，重新新建新的队列，旧的队列自动删除）
                }
            );
          },
          (err) => {
            //第一次连接失败和连接后断开连接都会调用这个函数 此处调用重连，倒计时接口
            console.log("自检倒计时重连错误信息：", err);
            // this.crewCountdownTime();
            reconnect();
          }
      );
    }

    //ws断线重连
    function reconnect() {
      console.log(lockReconnect)
      if (lockReconnect) return;
      lockReconnect = true;
      //没连接上会一直重连，设置延迟避免请求过多
      setTimeout(function () {
        (lockReconnect = false),
            console.log("自检倒计时开始重连"),
            connection(),
            console.log("自检倒计时重连结束");
      }, 3000);
    }

    function disconnect() {
      if (stompClient != null) {
        stompClient.disconnect();
        console.log("Disconnected");
      }
    }

    // WebSocket 结束
    return {
      count,
      connection,
      lockReconnect,
      disconnect,
      getCountdownData
    }
  },

  mounted() {
    console.log(this.count) // 0
    // 连接
    this.connection();
    // 第一次进页面主动查询
    this.getCountdownData();
  }

}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}

ul {
  list-style-type: none;
  padding: 0;
}

li {
  display: inline-block;
  margin: 0 10px;
}

a {
  color: #42b983;
}
</style>
