<template>
  <div id="app">
    <div class="userWrap" v-if="!joined">
      <h3>Enter username and secret key to join chat...</h3>
      <input placeholder="username" class="usernameInput" v-model="username">
      <input placeholder="key" type="password" number class="secretInput" v-model="secret">
      <button @click="joinChat" class="join">Join</button>
    </div>

    <div class="cypherWrap" v-if="joined">
      <select class="selectUser" v-model="selectedUser" @change="changeSelectedUser">
        <option value="">Select user to send encrypted message</option>
        <option v-for="user in users">{{user.username}}</option>
      </select>
    </div>

    <div class="chatArea">

      <div class="textArea">
        <div class="messagesWrap" :class="msg.user==username?'left':'right'" v-for="msg in messages">
          <h3>{{msg.user}}</h3>
          <hr>
          <p>{{msg.msg}}</p>
        </div>
      </div>

      <div class="userArea" v-if="joined">
        <div class="msgWrap">
          <h3 class="username">{{username}}</h3>
          <input class="msgInput" @keypress.enter="sendMsg" @keyup="putEnc" v-model="msg" placeholder="Start typing...">
          <button @click="sendMsg" class="send">Send</button>
        </div>

        <transition-group name="list" tag="p">
          <span v-for="(item,i) in encMsg" :key="i" class="list-item">
            {{ item }}
          </span>
        </transition-group>

      </div>

      

    </div>

    <div class="title">
      <h1>Caesar Cipher Chat</h1>
    </div>

    



  </div>
</template>

<script>

export default {
  data(){
    return{
      username:'',
      joined:false,
      msg:'',
      encMsg:'',
      messages:[],
      users:[],
      selectedUser:'',
      tocak: 23,
      baza: 7,
      secretContacts:[],
      key:0,
      secret:null,
      letters : ['a','b','c','d','e','f','g','h','i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y','z',
                 'A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q','R','S','T','U','V','W','X','Y','Z',
                 ' ',',','.','`','1','2','3','4','5','6','7','8','9','0','-','=','_','+','?','!','@','#','$','%','^','&','*','(',')']
    }
  },
  methods:{
    changeSelectedUser(){
      let obj = {
        to:this.selectedUser,
        sender:this.username,
        number:this.firstStep(this.secret)
      }  
      //console.log(obj);
      
      socket.emit('getSecret',obj);
      
    },
    firstStep(x){
      return Math.pow(this.baza,x)%this.tocak;
    },
    secondStep(x,a){
      return Math.pow(x,a)%this.tocak;
    },
    encr(text,key){
      var res = '';
        for(var i = 0;i<text.length;i++){
          var j = this.letters.indexOf(text[i]) + key;
            if(j>81) j -= 82;
          res += this.letters[j];
        }	
        return res;
    },
    decr(text,key){
      var res = '';
        for(var i = 0;i<text.length;i++){
          var j = this.letters.indexOf(text[i]) - key;
            if(j<0) j += 82;
          res += this.letters[j];
        }	
        return res;
    },
    joinChat(){
      if(this.username != '' && this.secret!=null){
        socket.emit('join',this.username);
        this.joined = true;
        socket.on('sendSecret.'+this.username,(obj) => {
         // console.log(obj.number,this.secret);
          obj.number = this.secondStep(obj.number,this.secret);
          //console.log(obj);
          this.secretContacts.push(obj);
          console.log(this.secretContacts);
          
          let myObj = {
            to:obj.from,
            from:this.username,
            number:this.firstStep(this.secret)
          }
          socket.emit('confirmSecret',myObj);
        })

        socket.on('confirmSecret.'+this.username,(obj) => {
         // console.log(obj.number,this.secret);
          obj.number = this.secondStep(obj.number,this.secret);
          //console.log(obj);
          let myObj = {
            from:obj.from,
            number:obj.number
          }
          this.secretContacts.push(myObj);
          console.log(this.secretContacts);
          
        })
      }
        
    },
    putEnc(){
      this.encMsg = this.msg;
    },
    sendMsg(){
      for(let i = 0; i < this.secretContacts.length; i++){
        if(this.secretContacts[i].from == this.selectedUser)
            this.key = this.secretContacts[i].number;
      }

    if(this.key!=0){
        let arr = this.encr(this.msg,this.key);
        this.encMsg = '';
        for (let i = 0; i < arr.length; i++) {
          setTimeout(() => {
            this.encMsg += arr[i];
          }, i*50);
        }

        setTimeout(() => {
            socket.emit('chat',{user:this.username,msg:this.encr(this.msg,this.key)});
            this.msg = '';
            this.encMsg = '';
          }, arr.length*50);

      }else{
        socket.emit('chatPublic',{user:this.username,msg:this.msg});
        
        this.msg = '';
        this.encMsg = '';
      }
      
      
        
        
      
    },
    sendData() {
      socket.emit('leave',this.username);
    },
  },
  mounted(){
    socket.on('getUsers', (users) => {
      for(let user in users){
        if(user == this.username)
          delete users[user];
      }
      this.users = users;
    });
    socket.on('chat', (payload) => {
      for(let i = 0; i < this.secretContacts.length; i++){
          if(this.secretContacts[i].from == payload.user)
            this.key = this.secretContacts[i].number;
        }
      payload.msg = this.decr(payload.msg,this.key);
      this.messages.push(payload);
      this.key = 0;
    })

    socket.on('chatPublic', (payload) => {
      this.messages.push(payload);
      this.key = 0;
    })
    
  },
  created() {
    window.addEventListener('beforeunload', this.sendData)
  },
  
}
</script>

<style lang="scss">
*,html{
  margin:0;
  padding:0;
  box-sizing: border-box;
  outline:0;
}
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color:#948b8b;
  margin-top: 10px;
}
.chatArea{
  margin:10px 30px;
  min-width: 80vw;
  height: 80vh;
  background-color: rgba(27, 86, 158, 0.151);
  border:1px solid black;
  border-radius:0 15px;
  position:relative;
  overflow: auto;
}
.cypherWrap{
  margin: 10px 30px;
  min-width: 80vw;
  height: 10vh;
  background-color: #3b75e066;
  border:1px solid black;
  border-radius: 0 15px;
}
.userArea{
  font-size: 20px;
  position:absolute;
  bottom:5%;
  width: 100%;
}
.msgInput{
  border: 1px solid white;
  font-size: 25px;
  width: 60%;
  background-color: rgba(0, 67, 255, 0.18);
  padding-left: 5px;
  outline:0;
}
.usernameInput{
    border: 1px solid white;
    font-size: 30px;
    width: 30%;
    background-color: rgba(0, 67, 255, 0.18);
    padding: 10px;
    margin-right: 15px;
    border-radius: 0 15px;
    outline: 0;
}
.secretInput{
    border: 1px solid white;
    font-size: 30px;
    width: 10%;
    background-color: rgba(0, 67, 255, 0.18);
    padding: 10px;
    margin-right: 15px;
    border-radius: 0 15px;
    outline: 0;
}
button{
  border: 1px solid white;
    font-size: 30px;
    width: 15%;
    border-radius:0 15px 0 0;
    padding: 10px;
    background-color: rgb(35, 137, 243);
    color: white;
    cursor:pointer;
    outline: 0;
}
.join{
  border-radius:0 15px;
}
.send{
  border-radius:0 15px 0 0;
}
.msgWrap{
  
  display: flex;
  flex-wrap: wrap;
  margin:0 30px;
  background-color: transparent;
  outline: 0;
}
.username{
  border: 1px solid white;
  font-size: 30px;
  width: 25%;
  border-radius: 0 0 0 15px;
  padding:10px;
  background-color: rgb(35, 137, 243);
  color: white;
  outline: 0;
}
.messagesWrap{
  margin:1% 2%;
  padding:1%;
  width:48%;
  border-radius:0 15px;
  text-align: left;
}
.left{
      background-color: #b1c8f3;
      color: black;
}
.right{
  background-color: #2389f3;
    -webkit-transform: translate(100%, 0);
    transform: translate(100%, 0);
    color: white;
}
.selectUser{
  width:65%;
  margin: 20px;
  padding: 10px;
  font-size: 20px;
  text-align: center;
}
.title{
  margin: 20px;
  font-size: 30px;
}
.list-item {
  display: inline-block;
  margin-right: 10px;
}
.list-enter-active, .list-leave-active {
  transition: all 1s;
}
.list-enter, .list-leave-to /* .list-leave-active below version 2.1.8 */ {
  opacity: 0;
  transform: translateY(30px);
}
</style>
