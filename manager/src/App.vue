<script setup>
  import { ref, onMounted, computed } from 'vue';
  import { notify } from "@kyvg/vue3-notification";
  import Modal from 'bootstrap/js/dist/modal'
  const Data = ref([]);
  const currentTr= ref(0);
  const currentData = ref([]);
  const mode = ref('');
  const searchId = ref('');
  const cancelData = ref({});
  const modal = ref();
  let myModal = '';
  const create = onMounted(()=>{
    myModal = new Modal(modal.value);
    getNotCheckedData();
  })

  const getNotCheckedData = ()=>{
    Data.value = [];
    mode.value = 'new';
    const today = new Date();
    fetch("https://doritos-event.com:8008/POST_GetNotCheckedList", {
      method: 'POST',
      headers: {
        "Content-Type": `application/json`,
      },
      body: JSON.stringify({
        "Day": today,
      })
    })
      .then(response => response.json())
      .then(result => {
        console.log(result)
        Data.value = result.NotCheckedList;
      })
      .catch(error => console.log('error', error));
  }
  const getCheckedData = ()=>{
    Data.value = [];
    mode.value = 'checked';
    const today = new Date();
    fetch("https://doritos-event.com:8008/POST_GetCheckedList", {
      method: 'POST',
      headers: {
        "Content-Type": `application/json`,
      },
      body: JSON.stringify({
        "Day": today,
      })
    })
      .then(response => response.json())
      .then(result => {
        Data.value = result.CheckedList;
      })
      .catch(error => console.log('error', error));
  }
  const getCancelData = ()=>{
    Data.value = [];
    mode.value = 'cancel';
    const today = new Date();
    fetch("https://doritos-event.com:8008/POST_GetCancelList", {
      method: 'POST',
      headers: {
        "Content-Type": `application/json`,
      },
      body: JSON.stringify({
        "Day": today,
      })
    })
      .then(response => response.json())
      .then(result => {
        Data.value = result.CancelList;
      })
      .catch(error => console.log('error', error));
  }

  const handleClick = (index)=>{      
    currentTr.value = index;
  }
  const dateCompute = (dateString, target)=>{
    let date = new Date(dateString);
    if(target == 'date') return date.toLocaleDateString()
    else if(target == 'time') return date.toLocaleTimeString()
  }
  const dataSplice = (i) =>{
    Data.value.splice(i, 1);
  } 

  const gocheck = (data, i) =>{
    let regex = /^\d{2}-\d{2}[a-zA-Z]{1}$/;
    if(!regex.test(data.Order)) notify({ type: "error", text: `請正確填寫梯次`,duration: 5000 })
    else {
      data.Order = data.Order.toUpperCase();
      fetch("https://doritos-event.com:8008/POST_Check", {
        method: 'POST',
        headers: {
          "Content-Type": `application/json`,
        },
        body: JSON.stringify({
          "ID": data.ID,
          "Order": data.Order
        })
      })
        .then(response => response.json())
        .then(result => {
          if(result.Result) {
            dataSplice(i);
            notify({ type: "success", text: `${data.Order} ${data.NationalID} 已審核`,duration: 5000 });
          }
          else {
            notify({ type: "error", text: `審核失敗，梯次已存在`,duration: 5000 });
          }
        })
        .catch(error => console.log('error', error));
    }
  }
  const uncheck = (data, i) =>{
    if (confirm('確定要退審 '+data.NationalID+' 嗎?') == true) {
      fetch("https://doritos-event.com:8008/POST_ReturnCheck", {
        method: 'POST',
        headers: {
          "Content-Type": `application/json`,
        },
        body: JSON.stringify({
          "ID": data.ID,
        })
      })
        .then(response => response.json())
        .then(result => {
          if(result.Result) {
            dataSplice(i);
            notify({ type: "success", text: `${data.NationalID} 已退審`,duration: 5000 });
          }
          else {
            notify({ type: "error", text: `退審失敗`,duration: 5000 });
          }
        })
        .catch(error => console.log('error', error));
    }
  }

  const dataFilter = computed(()=>{
    return Data.value.filter(data=>data.NationalID.match(searchId.value))
  })

  const cancelTemp = ref({});
  const gocancel = (data, index)=>{
    cancelData.value = data;
    cancelData.value.index = index;
    cancelTemp.value = {};
    myModal.show();
  }
  const checkcancel = ()=>{
    if(cancelTemp.value.type&&cancelTemp.value.reason&&cancelTemp.value.doubleID) {
      if(cancelData.value.NationalID == cancelTemp.value.doubleID) {
        fetch("https://doritos-event.com:8008/POST_Cancel", {
          method: 'POST',
          headers: {
            "Content-Type": `application/json`,
          },
          body: JSON.stringify({
            "ID": cancelData.value.ID,
            "Type": Number(cancelTemp.value.type),
            "Reason": cancelTemp.value.reason
          })
        })
          .then(response => response.json())
          .then(result => {
            if(result.Result) {
              dataSplice(cancelData.value.index);
              myModal.hide();
              notify({ type: "success", text: `${cancelData.value.NationalID} 已取消`,duration: 5000 });
            }
            else {
              notify({ type: "error", text: `取消失敗`,duration: 5000 });
            }
          })
          .catch(error => console.log('error', error));
      }
      else {
        notify({ type: "error", text: `號碼填寫錯誤`,duration: 5000 });
      }
    }
    else {
      notify({ type: "error", text: `請正確填寫欄位`,duration: 5000 });
    }
  }
</script>

<template>
  <div class="container vh100">
    <div class="d-flex justify-content-between pt-3">
      <ul class="nav">
        <li class="nav-item">
          <button class="px-3 py-2 btn btn-success box_shadow nav_btn text-white" :class="{'btn_mode': mode == 'new'}" @click="getNotCheckedData">新名單</button>
        </li>
        <li class="nav-item">
          <button class="px-3 py-2 btn btn-warning box_shadow nav_btn text-white" :class="{'btn_mode': mode == 'checked'}" @click="getCheckedData">已審核</button>
        </li>
        <li class="nav-item">
          <button class="px-3 py-2 btn btn-danger box_shadow nav_btn text-white" :class="{'btn_mode': mode == 'cancel'}" @click="getCancelData">取消</button>
        </li>
      </ul>
      <div class="col-3">
        <input type="text" class="form-control" id="search" placeholder="search" v-model.trim="searchId">
      </div>
    </div>
    <div class="p-3 content" :class="{'bg-success': mode == 'new','bg-warning': mode == 'checked','bg-danger': mode == 'cancel'}">
      <div class="h-100 overflow-auto">
        <table class="table table-bordered table_active mb-0" >
          <thead>
            <tr class="table-dark border-white table_sticky">
              <th scope="col" class="text-center" v-if="mode != 'cancel'">審核</th>
              <th scope="col" class="text-center" v-if="mode == 'cancel'">類別</th>
              <th scope="col" class="text-center">梯次<span class="ms-1" style="font-size: 12px;" v-if="mode == 'new'">( ＿＿-＿＿▲ )</span></th>
              <th scope="col" class="text-center">姓名</th>
              <th scope="col" class="text-center">身份證字號</th>
              <th scope="col" class="text-center">連絡電話</th>
              <th scope="col" class="text-center">年次</th>
              <th scope="col" class="text-center" v-if="mode == 'new'">填寫時間</th>
              <th scope="col" class="text-center" v-if="mode != 'cancel'">取消</th>
              <th scope="col" class="text-center" v-if="mode == 'cancel'">原因</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(data,i) in dataFilter" :key="i" :class="{'table_gray':i%4<2,'activeLi': currentTr == i}" class="border-white table_white"  @click="handleClick(i)">
              <th class="align-middle text-center" v-if="mode != 'cancel'">
                <button type="button" class="btn btn-warning" @click="gocheck(data, i)" v-if="mode == 'new'">審核</button>
                <button type="button" class="btn btn-success" @click="uncheck(data, i )" v-else-if="mode == 'checked'">退審</button>
              </th>
              <th class="align-middle text-center" v-if="mode == 'cancel'">
                {{ data.Type==0?'其他':data.Type==1?'身體不適':data.Type==2?'受傷':data.Type==3?'有事忙碌':'資料有誤' }}
              </th>
              <td class="align-middle text-center">
                <input type="text" class="text_w text-center" v-model="data.Order" v-if="mode == 'new'">
                <input type="text" class="text_w text-center" v-model="data.Order" v-else disabled>
              </td>
              <td class="align-middle text-center">{{ data.Name }}</td>
              <td class="align-middle text-center">{{ data.NationalID }}</td>
              <td class="align-middle text-center">{{ data.Phone }}</td>
              <td class="align-middle text-center" :class="{'bg-danger text-white':data.YearOfBirth-1911==101||data.YearOfBirth-1911==58}">{{ data.YearOfBirth-1911 }}</td>
              <td class="align-middle text-center" v-if="mode == 'new'">{{ dateCompute(data.SignUpTime, 'time') }}</td>
              <th class="align-middle text-center" v-if="mode != 'cancel'">
                <button type="button" class="btn btn-danger" @click="gocancel(data, i)">取消</button>
              </th>
              <td class="align-middle text-center" v-if="mode == 'cancel'">{{ data.Reason }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
    <notifications position="bottom right"/>
  </div>
  <div class="modal fade" id="staticBackdrop" data-bs-backdrop="static" data-bs-keyboard="false" tabindex="-1" aria-labelledby="staticBackdropLabel" aria-hidden="true" ref="modal">
    <div class="modal-dialog">
      <div class="modal-content">
        <div class="modal-header">
          <h5 class="modal-title" id="staticBackdropLabel">取消報名</h5>
        </div>
        <div class="modal-body">
          <div class="mb-3">
            姓名：{{cancelData.Name}}<span class="d-block">身份證號碼：{{cancelData.NationalID}}</span>
          </div>
          <div class="mb-3">
            <label for="type" class="form-label">類別<span class="text-danger ms-1">( 必填 )</span></label>
            <select class="form-select" id="type" v-model="cancelTemp.type">
              <option selected disabled value=""></option>
              <option value="4">資料有誤：( id、年齡、姓名、電話 )</option>
              <option value="3">有事忙碌：( 久候棄權等 )</option>
              <option value="2">受傷：( 外傷 )</option>
              <option value="1">身體不適：( 內傷｜中暑、月經等 )</option>
              <option value="0">其它：( 沒證件/沒臉書、資料/年齡不符、隊友不玩等 )</option>
            </select>
          </div>
          <div class="mb-3">
            <label for="description" class="form-label">事由<span class="text-danger ms-1">( 必填 )</span></label>
            <textarea type="text" class="form-control" id="description" placeholder="請填寫原因" rows="3" v-model="cancelTemp.reason"></textarea>
          </div>
          <div class="mb-3">
            <label for="double" class="form-label">再次確認身份證號碼</label>
            <input type="text" class="form-control" id="double" autocomplete="off" placeholder="輸入取消身份證號碼" v-model="cancelTemp.doubleID">
          </div>
        </div>
        <div class="modal-footer">
          <button type="button" class="btn btn-danger" @click="checkcancel">刪除</button>
          <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">取消</button>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss">
  @import "../node_modules/bootstrap/scss/bootstrap";
  .text_w{
    width: 100px;
    &:focus-visible{
      outline: none;
    }
  }
  .activeLi{
    background-color: rgb(0, 162, 255)!important;
    color: white;
  }
  .table_white{
    background-color: rgb(252, 252, 252);
  }
  .table_white.table_gray{
    background-color: rgb(219, 219, 219);
  }
  tbody > tr > th, tbody > tr > td {
    background-color: unset !important;
    color: unset !important;
  }
  .vue-notification > .notification-content {
    font-size: 16px;
    border-radius: 5px;
  }
  .box_shadow:focus, .box_shadow:active:focus{
    box-shadow: none;
  }
  .nav_btn{
    border-radius: 10px 10px 0 0;
  }
  .table > :not(:first-child) {
    border-top: 0;
  }
  .btn_mode{
    width: 100px;
  }
  .vh100{
    height: 100vh;
  }
  .content{
    height: calc(100% - 100px);
    border-radius: 0 10px 10px;
  }
  .table_sticky{
    position: sticky;
    top: -1px;
  }
  </style>