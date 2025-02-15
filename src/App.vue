<template>
  <div id="app">
    <h2>{{ isLogin ? '登入' : '註冊' }}</h2>

    <!-- 登入區塊 -->
    <form v-if="isLogin" @submit.prevent="handleLogin">
      <div>
        <label>手機號碼：</label>
        <input type="text" v-model="loginPhoneNumber" required>
      </div>
      <div>
        <label>密碼：</label>
        <input type="password" v-model="loginPassword" required>
      </div>
      <button type="submit">登入</button>
    </form>

    <!-- 註冊區塊 -->
    <form v-else @submit.prevent="handleRegister">
      <div>
        <label>使用者名稱：</label>
        <input type="text" v-model="registerUsername" required>
      </div>
      <div>
        <label>手機號碼：</label>
        <input type="text" v-model="registerPhoneNumber" required>
      </div>
      <div>
        <label>密碼：</label>
        <input type="password" v-model="registerPassword" required>
      </div>
      <button type="submit">註冊</button>
    </form>

    <button @click="isLogin = !isLogin">切換至{{ isLogin ? '註冊' : '登入' }}</button>
    
    <p v-if="message">{{ message }}</p>

    <hr>
    <h2>查詢庫存</h2>

    <!-- ISBN 輸入框 -->
    <input v-model="isbn" type="text" placeholder="請輸入 ISBN 編號" />

    <!-- 查詢按鈕 -->
    <button @click="fetchInventory">查詢</button>

    <!-- 顯示錯誤訊息 -->
    <p v-if="errorMessage" style="color: red;">{{ errorMessage }}</p>

    <!-- 顯示庫存結果 -->
    <div v-if="inventoryList.length">
      <h3>庫存結果：</h3>
      <table class="table">
        <thead>
          <tr>
            <th>庫存 ID</th>
            <th>ISBN</th>
            <th>狀態</th>
            <th>存入時間</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="item in inventoryList" :key="item.inventoryId">
            <td>{{ item.inventoryId }}</td>
            <td>{{ item.isbn }}</td>
            <td>{{ item.status }}</td>
            <td>{{ formatDate(item.storeTime) }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <hr>
    <h2>借書記錄查詢</h2>

    <!-- 是否篩選已歸還 -->
    <label>
      <input type="checkbox" v-model="isReturned" /> 僅顯示已歸還
    </label>

    <!-- 查詢按鈕 -->
    <button @click="fetchRecords">查詢記錄</button>

    <!-- 顯示錯誤訊息 -->
    <p v-if="errorMessage2" style="color: red;">{{ errorMessage2 }}</p>

    <!-- 顯示借書記錄 -->
    <div v-if="records.length">
      <h3>借書記錄：</h3>
      <table class="table">
        <thead>
          <tr>
            <th>記錄 ID</th>
            <th>使用者Id</th>
            <th>庫存Id</th>
            <th>借閱時間</th>
            <th>歸還時間</th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="record in records" :key="record.recordId">
            <td>{{ record.recordId }}</td>
            <td>{{ record.userId }}</td>
            <td>{{ record.inventoryId }}</td>
            <td>{{ formatDate(record.borrowingTime) }}</td>
            <td>{{ record.returnTime ? formatDate(record.returnTime) : '未歸還' }}</td>
          </tr>
        </tbody>
      </table>
    </div>

    <hr>
    <h2>借還書</h2>

    <div>
      <label>借書：</label>
          <!-- 輸入欄位讓使用者輸入 inventoryId -->
        <div>
          <label for="inventoryId">輸入 inventoryId：</label>
          <input type="text" v-model="newInventoryId" placeholder="輸入 inventoryId" />
          <button @click="addInventoryId">加入</button>
          <!-- 清除輸入的按鈕 -->
          <button @click="clearInventoryIds" :disabled="inventoryIds.length === 0">清除輸入</button>

        </div>

        <!-- 顯示已加入的 inventoryId 列表 -->
        <div v-if="inventoryIds.length > 0">
          <h3>已選擇的庫存 ID：</h3>
          <ul>
            <li v-for="id in inventoryIds" :key="id">{{ id }}</li>
          </ul>
        </div>

        <!-- 借書按鈕 -->
        <button @click="borrowBooks" :disabled="inventoryIds.length === 0">借書</button>

        <p v-if="errorMessage3" style="color: red;">{{ errorMessage3 }}</p>
    </div>
    <div>
      <label>還書：</label>
          <!-- 輸入欄位讓使用者輸入 recordId -->
        <div>
          <label for="recordId">輸入 recordId：</label>
          <input type="text" v-model="newRecordId" placeholder="輸入 recordId" />
          <button @click="addRecordId">加入</button>
          <!-- 清除輸入的按鈕 -->
          <button @click="clearRecordIds" :disabled="recordIds.length === 0">清除輸入</button>

        </div>

        <!-- 顯示已加入的 inventoryId 列表 -->
        <div v-if="recordIds.length > 0">
          <h3>已選擇的紀錄 ID：</h3>
          <ul>
            <li v-for="id in recordIds" :key="id">{{ id }}</li>
          </ul>
        </div>

        <!-- 借書按鈕 -->
        <button @click="returnBooks" :disabled="recordIds.length === 0">還書</button>

        <p v-if="errorMessage4" style="color: red;">{{ errorMessage4 }}</p>
    </div>
   
    

  </div>
</template>

<script>
  import { ref } from 'vue';
  import axios from 'axios';

  export default {
    setup() {
      const isLogin = ref(true);
      //登入變數
      const loginPhoneNumber = ref('');
      const loginPassword = ref('');
      
      //註冊變數
      const registerUsername = ref('');
      const registerPhoneNumber = ref('');
      const registerPassword = ref('');

      const message = ref('');

      //查詢庫存變數
      const isbn = ref(""); // 使用者輸入的 ISBN
      const inventoryList = ref([]); // 查詢結果
      const errorMessage = ref(""); // 錯誤訊息

      //查詢借閱紀錄變數
      const records = ref([]); // 借書記錄
      const isReturned = ref(false); // 是否篩選已歸還
      const errorMessage2 = ref("");// 錯誤訊息

      //借還書變數
      const newInventoryId = ref(''); // 新輸入的 inventoryId
      const inventoryIds = ref([]); // 儲存使用者輸入的所有 inventoryId
      const errorMessage3 = ref('');  // 顯示訊息

      const newRecordId = ref(""); // 新輸入的 recordId
      const recordIds = ref([]); // 儲存使用者輸入的所有 recordId
      const errorMessage4 = ref(""); // 顯示訊息



      //登入動作
      const handleLogin = async () => {
        const url = 'http://localhost:8080/users/login';
        const payload = {
          phoneNumber: loginPhoneNumber.value,
          password: loginPassword.value
        };

        try {
          const response = await axios.post(url, payload);
          message.value = '登入成功';

          //儲存回傳token&userId
          localStorage.setItem('token', response.data.token);
          localStorage.setItem('userId', response.data.userId);
        } catch (error) {
          message.value = error.response?.data?.message || '登入失敗';
        }
      };

      //註冊動作
      const handleRegister = async () => {
        const url = 'http://localhost:8080/users/register';
        const payload = {
          userName: registerUsername.value,
          phoneNumber: registerPhoneNumber.value,
          password: registerPassword.value
        };

        try {
          await axios.post(url, payload);
          message.value = '註冊成功，請登入';
          isLogin.value = true; // 註冊成功後切換至登入畫面
        } catch (error) {
          message.value = error.response?.data?.message || '註冊失敗';
        }
      };

      //查詢庫存
      const fetchInventory = async () => {
        if (!isbn.value.trim()) {
          errorMessage.value = "請輸入 ISBN 編號";
           return;
         }

        try {
          const response = await axios.get(`http://localhost:8080/inventories/${isbn.value}`);
          inventoryList.value = response.data;
          errorMessage.value = "";
        } catch (error) {
          console.error("發生錯誤:", error);
          errorMessage.value = "發生錯誤，請重試";
          inventoryList.value = [];
        }
      };

      //格式化日期
      const formatDate = (dateString) => {
        return new Date(dateString).toLocaleString();
      };

      // 查詢借書記錄
      const fetchRecords = async () => {
        const userId = localStorage.getItem('userId');  // 取出 userId
        const token = localStorage.getItem('token');    // 取出 Token

        if (!token || !userId) {
          errorMessage2.value = "請先登入";
          return;
        }

        try {
          const response = await axios.get(`http://localhost:8080/users/${userId}/records`, {
            headers: { Authorization: `Bearer ${token}` },
            // params: { isReturned: isReturned.value }
          });

          records.value = response.data;
          errorMessage2.value = "";
        } catch (error) {
          console.error("查詢記錄錯誤:", error);
          errorMessage2.value = "查詢失敗，請重試";
        }
      };

      //借書
      //新增 inventoryId
      const addInventoryId = () => {
        const id = newInventoryId.value.trim();
        if (id && !inventoryIds.value.includes(id)) {
          inventoryIds.value.push(id);  // 加入 inventoryId
          newInventoryId.value = '';  // 清空輸入框
        }
      };
      //清除所有輸入的 inventoryId
      const clearInventoryIds = () => {
        inventoryIds.value = [];  //清空inventoryIds
      };
      const borrowBooks = async () => {
        const userId = localStorage.getItem('userId');  // 取出 userId
        const token = localStorage.getItem('token');    // 取出 Token

        if (!userId || !token) {
          errorMessage3.value = '請先登入';
          return;
        }

        const url = `http://localhost:8080/users/${userId}/borrow`;
        const payload = {
          borrowBookList: inventoryIds.value.map(id => ({ inventoryId: id }))
        };

        try {
          await axios.post(url, payload, {
            headers: { Authorization: `Bearer ${token}` }
          });

          errorMessage3.value = '借書成功';
          clearInventoryIds();// 借書後清空選擇紀錄
        } catch (error) {
          errorMessage3.value = error.response?.data?.errorMessage3 || '借書失敗';
        }
      };

      //還書
      //新增recordIds
      const addRecordId = () => {
        const id = newRecordId.value.trim();
        if (id && !recordIds.value.includes(id)) {
          recordIds.value.push(id);
        }
        newRecordId.value = ""; // 清空輸入框
      };
      //清除所有輸入 recordId
      const clearRecordIds = () => {
        recordIds.value = [];
      };
      const returnBooks = async () => {
        const userId = localStorage.getItem('userId');  // 取出 userId
        const token = localStorage.getItem('token');    // 取出 Token

        if (!userId || !token) {
          errorMessage4.value = '請先登入';
          return;
        }

        const url = `http://localhost:8080/users/${userId}/return`;

        const payload = {
          recordIdList: recordIds.value.map(id => parseInt(id))
        };

        try {
          await axios.post(url, payload, {
            headers: { Authorization: `Bearer ${token}` }
          });

          errorMessage4.value = "還書成功";
          clearRecordIds(); // 還書後清空選擇紀錄
        } catch (error) {
          errorMessage4.value = error.response?.data?.errorMessage4 || "還書失敗";
        }
      };

      return {
        isLogin,
        loginPhoneNumber,
        loginPassword,
        registerUsername,
        registerPhoneNumber,
        registerPassword,
        message,
        isbn,
        inventoryList,
        errorMessage,
        records, 
        isReturned, 
        errorMessage2,
        newInventoryId,
        inventoryIds,
        errorMessage3, 
        newRecordId,
        recordIds,
        errorMessage4,
        handleLogin,
        handleRegister,
        fetchInventory,
        fetchRecords,
        formatDate,
        addInventoryId,
        borrowBooks,
        clearInventoryIds,
        addRecordId,
        clearRecordIds,
        returnBooks
      };
    }
  };
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
