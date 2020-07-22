<template>
  <div class="hello">
    <h1>{{ msg }}</h1>
    <p>残金：{{availableCost().toLocaleString()}}円 (使用した金額：{{sumCost().toLocaleString()}}円)</p>

    <el-table
      :data="dataList"
      :default-sort = "{prop: 'strDate', order: 'descending'}"
      class="table"
      height="420px"
      @cell-click="rowClick">
      <el-table-column
        prop="strDate"
        label="日付"
        width="60px"
        padding="0">
      </el-table-column>
      <el-table-column
        prop="menu"
        label="メニュー">
      </el-table-column>
      <el-table-column
        prop="cost"
        label="金額"
        width="60px">
      </el-table-column>
      <el-table-column width="60px">
          <el-button type="danger" icon="el-icon-delete" size="mini" circle></el-button>
      </el-table-column>
    </el-table>

    <el-button @click="plusClick" icon="el-icon-plus" circle></el-button>

    <el-dialog
      title="メシ"
      witdh="90%"
      :visible.sync="dialogVisible"
      :before-close="closeDialog">
      <ul>
        <li class="dialog-contents">
          <p class="dialog-contents-label">日付:</p>
          <el-date-picker
            v-model="dialogData.date"
            type="date"
            placeholder="Pick a date"
            class="dialog-contents-input">
          </el-date-picker>
        </li>
        <li class="dialog-contents">
          <p class="dialog-contents-label"></p>
          <el-radio-group class="dialog-contents-radio" v-model="dialogData.timing" size="small">
            <el-radio-button label="朝" ></el-radio-button>
            <el-radio-button label="昼"></el-radio-button>
            <el-radio-button label="夜"></el-radio-button>
            <el-radio-button label="間食"></el-radio-button>
          </el-radio-group>
        </li>
        <li class="dialog-contents">
          <p class="dialog-contents-label">メニュー:</p>
          <el-input class="dialog-contents-input" v-model="dialogData.menu"></el-input>
        </li>
        <li class="dialog-contents">
          <p class="dialog-contents-label">金額:</p>
          <el-input-number class="dialog-contents-input" v-model="dialogData.cost" :min="1" :max="10000"></el-input-number>
        </li>
      </ul>
      <span slot="footer" class="dialog-footer">
        <el-button @click="closeDialog()">Cancel</el-button>
        <el-button type="primary" @click="register()">OK</el-button>
      </span>
    </el-dialog>

  </div>
</template>

<style>
  @import "../assets/css/list.css";
</style>

<script src="https://www.gstatic.com/firebasejs/7.14.5/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/7.14.5/firebase-auth.js"></script>

<script>
import _ from 'lodash'
const LIMIT_COST = 100000

const firebase = require("firebase");
require("firebase/firestore");

var firebaseConfig = {
  apiKey: "AIzaSyB-s6Is58X80cuBbhXCg5Z3JQ153lVIYwk",
  authDomain: "my-kyuuhukin.firebaseapp.com",
  databaseURL: "https://my-kyuuhukin.firebaseio.com",
  projectId: "my-kyuuhukin",
  storageBucket: "my-kyuuhukin.appspot.com",
  messagingSenderId: "756216179995",
  appId: "1:756216179995:web:64e601cfc85e82d02db7be",
  measurementId: "G-GB4RF4GYL9"
};

firebase.initializeApp(firebaseConfig);
var db = firebase.firestore();
var dbCollection = db.collection("kyuuhukin");

let isLogin = false;

firebase.auth().onAuthStateChanged(user => {
  if (user) {
    // ユーザーがログインしています。
    console.log("ログインなう：", user)
    isLogin = true;
  } else {
    // ユーザーはログインしていません。
    console.log("ログインしていないなう")
    isLogin = false;
  }
});

var provider = new firebase.auth.GoogleAuthProvider();

export default {
  name: 'HelloWorld',
  data () {
    return {
      msg: '給付金10万を使ってみる',
      dbDataList: null,
      dataList: [],
      dialogVisible: false,
      dialogData: {},
      LIMIT_COST: LIMIT_COST
    }
  },
  computed: {
  },
  methods: {
    zeroPadding (val) {
      if (String(val).length === 1) {
        val = '0' + val
      }
      return val
    },
    dateFormatMMDD (val) {
      const date = new Date(val)
      let month = this.zeroPadding(date.getMonth() + 1)
      let day = this.zeroPadding(date.getDate())
      return month + '/' + day
    },
    rowClick (row, column, cell, event) {
      // console.log("row: ", JSON.stringify(row))
      if (cell.getElementsByTagName('button').length === 0) {
        this.openEditDialog(row)
      } else {
        this.openDeleteDialog(row)
      }
    },
    plusClick () {
      this.dialogData = {
        id: null,
        date: new Date(),
        timing: '朝',
        menu: '',
        cost: '800',
        deleted: false
      }
      this.dialogVisible = true
    },
    openEditDialog (val) {
      let tmpVal = _.cloneDeep(val)
      tmpVal.date = tmpVal.date.toDate()
      this.dialogData = tmpVal
      this.dialogVisible = true
      // console.log(JSON.stringify(this.dialogData))
    },
    openDeleteDialog (val) {
      this.$confirm(this.dateFormatMMDD(val.date.toDate()) + ' "' + val.menu + '" を削除してよいか')
        .then(_ => {
          this.deleteDB(val)
        }).catch(_ => {})
    },
    closeDialog (done) {
      this.getDbDataList()
      if (done == null) {
        this.dialogVisible = false
      }else {
        done()
      }
    },
    sumCost () {
      let sum = 0
      for (let data of this.dataList) {
        sum += Number(data.cost)
      }
      return sum
    },
    availableCost () {
      let ret = LIMIT_COST - this.sumCost()
      if (ret < 0) {
        ret = 0
      }
      return ret
    },
    register() {
      // console.log(JSON.stringify(this.dialogData))
      if(this.dialogData.menu == '') {
        alert('メニューを入力してください')
        return
      }
      if(this.dialogData.id == null) {
        // let maxId = 0
        // this.dbDataList.forEach((doc) => {
        //   let data = doc.data()
        //   if (maxId < data.id) {
        //     maxId = data.id
        //   }
        // });
        // this.dialogData.id = maxId + 1
        this.registerDB(this.dialogData)
      } else{
        this.updateDB(this.dialogData)
      }
    },
    registerDB(data) {
      let _ = this
      dbCollection.add(data).then(function(docRef) {
          // console.log("Document written with ID: ", docRef.id);
          _.dialogVisible = false
          _.getDbDataList();
      })
      .catch(function(error) {
        alert(error)
          // console.error("Error adding document: ", error);
      });
    },
    getDbDataList() {
      let tmpList = []
      dbCollection.get().then((querySnapshot) => {
        this.dbDataList = querySnapshot
        querySnapshot.forEach((doc) => {
          let data = doc.data()
          data['strDate'] = this.dateFormatMMDD(data.date.toDate())
          data['id'] = doc.id
          tmpList.push(data)
        });
        this.dataList = tmpList
      });
    },
    updateDB(val) {
      let _ = this
      dbCollection.doc(val.id).update({
        date: val.date,
        timing: val.timing,
        menu: val.menu,
        cost: val.cost,
        deleted: val.deleted
      })
      .then(function() {
        // console.log("Document successfully updated!");
        _.closeDialog(null)
      })
      .catch(function(error) {
        alert(error)
        // console.error("Error updating document: ", error);
      });
    },
    deleteDB(val) {
      let _ = this
      dbCollection.doc(val.id).delete().then(function() {
        // console.log("Document successfully deleted! : " + val.id)
        _.getDbDataList()
      }).catch(function(error) {
        alert(error)
          // console.error("Error removing document: ", error)
      });
    }
  },
  created () {

    this.getDbDataList();

    // let _ = this;

    // if(isLogin) {
    //   this.getDbDataList();
    // } else {
    //   firebase.auth().signInWithPopup(provider).then(function(result) {
    //       // This gives you a Google Access Token. You can use it to access the Google API.
    //       var token = result.credential.accessToken;
    //       // The signed-in user info.
    //       var user = result.user;
    //       console.log(user)

    //       _.getDbDataList();
    //       // ...
    //     }).catch(function(error) {
    //       // Handle Errors here.
    //       var errorCode = error.code;
    //       var errorMessage = error.message;
    //       // The email of the user's account used.
    //       var email = error.email;
    //       // The firebase.auth.AuthCredential type that was used.
    //       var credential = error.credential;

    //       console.log(errorMessage)
    //       // ...
    //     });
    // }
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h1, h2 {
  font-weight: normal;
}
ul {
  list-style-type: none;
  padding: 0;
}
.table {
    display: inline-block;
    text-align: center;
    width: 100%;
}
.el-message-box {
    width: 250px;
}
.dialog-contents{
    display: inline-block;
    width: 100%;
}
li {
    margin: 20px 0 00px 0;
}
.dialog-contents-label{
    display: inline-block;
    width: 25%;
    margin: 0px 0 20px 0;
    text-align: left;
}
.el-date-editor.el-input{
    width: 60%;
}
.dialog-contents-input{
    display: inline-block;
    width: 60%;
}
.dialog-contents-radio{
    padding-left: 0px;
}
</style>
