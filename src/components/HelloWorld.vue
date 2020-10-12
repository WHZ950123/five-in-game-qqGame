<template>
  <div>
    <div class="head" v-if="!showQQGame">
      <el-input v-model="username" placeholder="请输入用户名"></el-input>
      <el-button type="primary" @click="toQQgame">确定</el-button>
      <!--<el-button type="primary" @click="init">确定</el-button>-->
    </div>
    <div v-if="showQQGame && !linkSuccess">
      <div>
        <el-button type="primary" @click="closeLink">退出游戏</el-button>
      </div>
      <div v-for="(item, index) in zhuozi" :key="index" class="zhuozi">
        <!-- 左边座位有人 -->
        <div v-if="item.user1">
          <img src="../assets/user.jpg" width="50px" height="50px" class="leftImg">
          <div class="enter_left">{{item.user1}}</div>
        </div>
        <!-- 左边座位无人 -->
        <div v-else>
          <img src="../assets/seat.jpg" width="50px" height="50px" class="leftImg" @click="enter(index)">
          <div class="enter_left" @click="enter(index)">加入</div>
        </div>
        <!-- 中间的棋盘 -->
        <img src="../assets/table.jpg" width="100px" height="100px" class="centerImg">
        <!-- 右边座位有人 -->
        <div v-if="item.user2">
          <img src="../assets/user.jpg" width="50px" height="50px" class="rightImg">
          <div class="enter_right">{{item.user2}}</div>
        </div>
        <!-- 右边座位无人 -->
        <div v-else>
          <img src="../assets/seat.jpg" width="50px" height="50px" class="rightImg" @click="enter(index)">
          <div class="enter_right" @click="enter(index)">加入</div>
        </div>
      </div>
    </div>
    <div v-if="linkSuccess">
      <div class="playPlace" @click="playGame($event)">
        <div v-for="(item, index) in qp" :key="index">
          <div
            v-for="(item2, index2) in item"
            :class="qp[index][index2] === -1 ? 'noQz' : qzClass[qp[index][index2]]"
            :style="{top: index * 30 + 20 + 'px', left: index2 * 30 + 20 + 'px'}"
            @click="sendPosition(index, index2)"
            :key="index2">
          </div>
        </div>
      </div>
      <div class="player">
        <div class="me">
          我：<br/>{{username}}<br/>
          {{!qzColor ? '' : (qzColor === 'black' ? '黑棋' : '白棋')}}
        </div>
        <div class="other">
          对手：<br/>{{otherName}}<br/>
          {{!qzColor ? '' : (qzColor === 'black' ? '白棋' : '黑棋')}}
        </div>
      </div>
      <div v-if="end" class="reBegin">
        <el-button v-if="otherName" type="primary" @click="reBegin">开始</el-button>
        <el-button class="end" type="primary" @click="seatUp">退出</el-button>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'HelloWorld',
  data () {
    return {
      path: 'ws:127.0.0.1:4000', //websocket连接地址
      socket: null, //websocket
      username: '', //用户名
      linkSuccess: false, //是否连接成功
      messageArr: [], //服务器发来的消息记录
      qzClass: { //棋子的class,0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
        0: 'blackQz',
        1: 'whiteQz',
        2: 'blackQz_just',
        3: 'whiteQz_just'
      },
      otherName: '', //对手的名字
      qp: [], //棋盘(二维数组)
      qzColor: '', //棋子颜色
      turnToYou: false, //是否该你下棋了
      lastPosition: { //上一个棋子的位置
        x: -1,
        y: -1,
        color: ''
      },
      end: true, //游戏是否结束
      zhuohaoNum: 3, //总桌号
      zhuozi: [], //桌子的数据,包含user1,user2
      showQQGame: false, //跳转到QQ游戏
      zhuohao: -1, //桌号
      hasSeat: false, //是否坐下了
      hasClickBegin: false //第一次坐下是否点击了开始
    }
  },
  mounted () {
    this.initQp()
    this.initZhuozi()
  },
  methods: {
    //重置棋盘
    initQp () {
      this.qp = []
      for (let i = 0; i < 15; i++) {
        let arr = []
        for (let j = 0; j < 15; j++) {
          arr.push(-1)
        }
        this.qp.push(arr)
      }
    },
    //初始化桌子
    initZhuozi() {
      for (let i = 0; i < this.zhuohaoNum; i++) {
        this.$set(this.zhuozi, i, {
          user1: '',
          user2: ''
        })
      }
    },
    //点击桌子加入游戏
    enter (index) {
      this.zhuohao = index
      this.socket.send(JSON.stringify({
        username: this.username,
        mes: 'sitdown',
        zhuohao: this.zhuohao
      }))
      this.hasSeat = true
      this.end = true
    },
    init () {
      if (typeof(WebSocket) === 'undefined') {
        console.error('您的浏览器不支持socket')
      } else{
        // 实例化socket
        this.socket = new WebSocket(this.path)
        // 监听socket连接
        this.socket.onopen = this.open
        // 监听socket错误信息
        this.socket.onerror = this.error
        // 监听socket消息
        this.socket.onmessage = this.getMessage
      }
    },
    open () {
      console.log("socket连接成功")
      this.socket.send(JSON.stringify({
        username: this.username,
        mes: 'link'
      }))
    },
    error () {
      console.log("连接错误")
    },
    getMessage (msg) {
      let data = JSON.parse(msg.data)
      console.log('服务端发来的消息:', data)
      const code = data.code
      const text = data.text
      if (code === 500) {
        this.$message.error(text)
        this.closeLink()
        this.linkSuccess = false
      } else {
        this.messageArr.push(data)
        if (code === 200) { //连接成功
          this.linkSuccess = true
        } else if (code === 202) { //服务器发过来的是对手的名字和棋子颜色
          this.otherName = text
          this.qzColor = data.qz
          if (this.qzColor === 'black') {
            this.turnToYou = true //黑棋先下
          } else {
            this.turnToYou = false
          }
        } else if (code === 204) { //发送棋子的位置
          const poArr = text.split(',')
          if (poArr[2] !== this.qzColor) { //服务器发来的颜色是对手的颜色,说明对手下棋了
            this.turnToYou = true
          }
          this.showOnQp(poArr[0], poArr[1], poArr[2])
        } else if (code === 206) { //发送桌子占用情况
          for (let item in this.zhuozi) {
            this.$set(this.zhuozi, item, {
              user1: '',
              user2: ''
            })
          }
          for (let item in text) {
            if (text[item].length > 0) {
              let man = {
                user1: text[item][0],
                user2: text[item][1]
              }
              this.$set(this.zhuozi, item, man)
            }
          }
          //如果有用户离开了桌子
          if (!this.zhuozi[this.zhuohao].user1 || !this.zhuozi[this.zhuohao].user2) {
            this.otherName = ''
            this.end = true
          }
        } else if (code === 208) { //发送对手名字,在2个玩家到齐后使用
          this.otherName = text
        }
      }
    },
    close () {
      console.log("socket已经关闭")
    },
    //跳转到QQ游戏
    toQQgame () {
      this.username = this.username.trim()
      if (!this.username) {
        this.$message.error('请输入用户名')
      } else {
        this.showQQGame = true
        this.init()
      }
    },
    //退出桌子
    seatUp () {
      this.socket.send(JSON.stringify({
        username: this.username,
        mes: 'situp',
        zhuohao: this.zhuohao
      }))
      //所有信息都清空或重置
      this.hasSeat = false
      this.end = false
      this.linkSuccess = false
      this.initQp()
      this.qzColor = ''
      this.otherName = ''
      this.turnToYou = false
      this.lastPosition.x = -1
      this.lastPosition.y = -1
      this.lastPosition.color = ''
    },
    closeLink () { //关闭连接
      /*
        readyState属性返回实例对象的当前状态，共有四种。
        CONNECTING：值为0，表示正在连接。
        OPEN：值为1，表示连接成功，可以通信了。
        CLOSING：值为2，表示连接正在关闭。
        CLOSED：值为3，表示连接已经关闭，或者打开连接失败。
      */
      if (this.socket && this.socket.readyState && this.socket.readyState !== 3) {
        this.socket.send(JSON.stringify({
          username: this.username,
          mes: 'close',
          zhuohao: this.zhuohao
        }))
        this.socket.close()
        console.log('关闭了连接')
      }
      //所有信息都清空或重置
      this.hasSeat = false
      this.end = false
      this.socket = null
      this.linkSuccess = false
      this.messageArr = []
      this.initQp()
      this.initZhuozi()
      this.showQQGame = false
      this.zhuohao = -1.
      this.qzColor = ''
      this.otherName = ''
      this.turnToYou = false
      this.lastPosition.x = -1
      this.lastPosition.y = -1
      this.lastPosition.color = ''
      console.log('socket:', this.socket)
    },
    playGame (event) { //点击棋盘(15*15)
      //位置是相对于页面左上角的点
      //一个格子大小为30px*30px
      //棋盘左上角的点的位置：clientX: 315, clientY: 35
      //棋盘右上角的点的位置：clientX: 740, clientY: 35
      //棋盘左下角的点的位置：clientX: 315, clientY: 460
      //棋盘右下角的点的位置：clientX: 740, clientY: 460
      console.log(event)
    },
    //将用户点击的坐标发送给服务器
    sendPosition (x, y) {
      if (this.qzColor <= 0) {
        return
      }
      if (!this.end) { //游戏未结束
        if (this.turnToYou) { //该你下棋了
          if (this.qp[x][y] !== -1) { //该坐标已经有棋子
            return
          }
          this.socket.send(JSON.stringify({
            username: this.username,
            mes: x + ',' + y + ',' + this.qzColor,
            zhuohao: this.zhuohao
          }))
          this.turnToYou = false
        } else {
          this.$message.error('莫慌,该对手了')
        }
      }
    },
    //服务器发来的棋子坐标,棋子颜色:0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
    showOnQp (x, y, qzColor) {
      if (this.lastPosition.x >= 0) { //将上一步的黑棋/白棋改成黑棋/白棋的样式
        let temp = this.qp[this.lastPosition.x]
        temp[this.lastPosition.y] = this.lastPosition.color === 'black' ? 0 : 1
        this.$set(this.qp, this.lastPosition.x, temp)
      }
      this.lastPosition.x = x
      this.lastPosition.y = y
      this.lastPosition.color = qzColor
      if (qzColor.length <= 0) {
        return
      } 
      //给二维数组设置值
      let temp = this.qp[x]
      temp[y] = qzColor === 'black' ? 2 : 3
      this.$set(this.qp, x, temp)
      //判断输赢
      const win = this.whoWin()
      if (win) {
        this.$message.success(win + '赢了')
        this.end = true
      }
    },
    //判断胜负
    //0-黑,1-白,-1-无,2-刚下的黑棋,3-刚下的白棋
    whoWin () {
      for (let i = 0; i < this.qp.length; i++) { //横着判断
        let row = this.qp[i] //每行的数据
        let blackNum = 0
        let whiteNum = 0
        for (let j = 0; j < row.length; j++) {
          let q = row[j]
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          if (blackNum === 5) {
            return '黑棋'
          } else if (whiteNum === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //竖着判断
        let blackNum = 0
        let whiteNum = 0
        for (let j = 0; j < 15; j++) {
          let q = this.qp[j][i]
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          if (blackNum === 5) {
            return '黑棋'
          } else if (whiteNum === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //左上角三角形的向右上判断
        let blackNum = 0
        let whiteNum = 0
        let blackNum2 = 0
        let whiteNum2 = 0
        for (let j = 0; j <= i; j++) {
          let q = this.qp[i-j][j] //左上角三角形的向右上判断
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          let q2 = this.qp[14-j][14-(i-j)] //右下角三角形的向右上判断
          if (q2 === -1) { //无棋子
            blackNum2 = 0
            whiteNum2 = 0
          } else if (q2 === 0 || q2 === 2) { //黑棋
            blackNum2++
            whiteNum2 = 0
          } else { //白棋
            whiteNum2++
            blackNum2 = 0
          }
          if (blackNum === 5 || blackNum2 === 5) {
            return '黑棋'
          } else if (whiteNum === 5 || whiteNum2 === 5) {
            return '白棋'
          }
        }
      }
      for (let i = 0; i < 15; i++) { //左下角三角形的向右下判断
        let blackNum = 0
        let whiteNum = 0
        let blackNum2 = 0
        let whiteNum2 = 0
        for (let j = 0; j < 15 - i; j++) {
          let q = this.qp[i+j][j] //左下角三角形的向右下判断
          if (q === -1) { //无棋子
            blackNum = 0
            whiteNum = 0
          } else if (q === 0 || q === 2) { //黑棋
            blackNum++
            whiteNum = 0
          } else { //白棋
            whiteNum++
            blackNum = 0
          }
          let q2 = this.qp[j][i+j] //右上角三角形的向右下判断
          if (q2 === -1) { //无棋子
            blackNum2 = 0
            whiteNum2 = 0
          } else if (q2 === 0 || q2 === 2) { //黑棋
            blackNum2++
            whiteNum2 = 0
          } else { //白棋
            whiteNum2++
            blackNum2 = 0
          }
          if (blackNum === 5 || blackNum2 === 5) {
            return '黑棋'
          } else if (whiteNum === 5 || whiteNum2 === 5) {
            return '白棋'
          }
        }
      }
    },
    //重新开始
    reBegin () {
      this.end = false
      this.initQp()
      this.qzColor = ''
      this.turnToYou = false
      this.lastPosition.x = -1
      this.lastPosition.y = -1
      this.lastPosition.color = ''
      this.socket.send(JSON.stringify({
        username: this.username,
        mes: 'reBegin',
        zhuohao: this.zhuohao
      }))
    },
  },
  destroyed () {
    // 销毁监听
    if (this.socket) {
      this.closeLink()
    }
  }
}
</script>

<style scoped>
.head {
  margin: 0 auto;
}
.el-input {
  width: 20%;
}
.playPlace {
  position: fixed;
  width: 450px;
  height: 450px;
  border: 1px solid red;
  top: 20px;
  left: 20px;
  background-image: url('../assets/five.jpg');
}
.player {
  position: fixed;
  top: 20px;
  left: 473px;
  width: 200px;
  height: 450px;
  border: 1px solid yellow;
}
.me {
  height: 223px;
  width: 100%;
  border: 1px solid green;
}
.other {
  height: 223px;
  width: 100%;
  border: 1px solid lightcoral;
}
.blackQz {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/black.png');
}
.whiteQz {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/white.png');
}
.noQz {
  width: 30px;
  height: 30px;
  position: fixed;
}
.blackQz_just {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/black_pick.png');
}
.whiteQz_just  {
  width: 30px;
  height: 30px;
  position: fixed;
  background-image: url('../assets/white_pick.png');
}
.reBegin {
  position: fixed;
  top: 480px;
  left: 20px;
}
.end {
  margin-left: 10px;
}
.zhuozi {
  width: 330px;
  height: 170px;
  border: 1px solid brown;
  float: left;
  margin: 10px 10px;
  position: static;
}
.enter_left {
  width: 80px;
  height: 30px;
  position: relative;
  left: 13px;
  top: 50px;
  border: 1px solid black;
  cursor: pointer;
  line-height: 30px;
}
.enter_right {
  width: 80px;
  height: 30px;
  position: relative;
  right: -235px;
  top: -140px;
  border: 1px solid black;
  cursor: pointer;
  line-height: 30px;
}
.leftImg {
  position: relative;
  left: -109px;
  top: 40px;
  border: 1px solid black;
  cursor: pointer;
}
.rightImg {
  position: relative;
  left: 110px;
  top: -150px;
  border: 1px solid black;
  cursor: pointer;
}
.centerImg {
  position: relative;
  left: 0px;
  top: -50px;
}
</style>