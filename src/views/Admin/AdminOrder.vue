<!--  
  问题：
  - 表格大小无法适配所有分辨率
  - 可以有新增新药品种类的功能
 -->

<template>
  <el-container>
    <el-main>
      <!-- 条件区 -->
      <div class="conditions">
        <center>
          <va-input v-model="filter" placeholder="请输入相关信息"></va-input>
        </center>
      </div>
      <!-- 条件区 -->

      <!-- 表格区 -->
      <div class="table">
        <va-data-table :items="tableItems" :columns="tableColumns" :filter="filter" :per-page="perPage"
          :current-page="curPage" :wrapper-size="550" hoverable clickable virtual-scroller noDataFilteredHtml="无数据"
          noDataHtml="无数据" @filtered="filteredCount = $event.items.length" @row:click="openModal">

          <!-- 分页 -->
          <template #bodyAppend>
            <br>
            <tr>
              <td colspan="6">
                <va-pagination v-model="curPage" :pages="pages" style="justify-content: center" :visible-pages="10" />
              </td>
            </tr>
          </template>
          <!-- 分页 -->

        </va-data-table>
      </div>
      <!-- 表格区 -->

      <!-- 弹窗 -->
      <div class="modal">
        <va-modal v-model="showModal" title="订单详情" ok-text="确认订单" cancel-text="取消操作" no-outside-dismiss @ok="confirm"
          @cancel="cancel">
          <div style="line-height: 200%;">订单号：{{ orderId }}</div>
          <va-data-table :items="modalItems" :columes="modalColumns" :wrapper-size="250" sticky-header hoverable
            virtual-scroller striped>
          </va-data-table>
        </va-modal>
      </div>
      <!-- 弹窗 -->

    </el-main>

  </el-container>
</template>

<script>
export default {
  data() {
    const perPage = 12;
    const tableColumns = ["订单号", "时间", "就诊人ID", "就诊人姓名", "金额", "状态"];
    const modalColumns = ["药品名", "数量", "单价", "总价"];

    return {
      // 筛选
      filter: "",
      filteredCount: 0,

      // 表格
      tableColumns,
      tableItems: [],

      // 分页
      perPage,
      curPage: 1,

      //弹窗
      showModal: false,
      selectedRow: 0,
      orderId: "",
      modalColumns,
      modalItems: [],

    }
  },

  computed: {
    // 页数
    pages() {
      return Math.ceil(this.filteredCount / this.perPage);
    }
  },

  methods: {
    // 刷新表格
    getTable() {
      fetch("http://124.223.143.21/api/Prescription/GetAll", {
        method: 'GET',
        redirect: 'follow'
      }).then(response => response.text())
        .then(result => {
          result = JSON.parse(result);
          this.tableItems = [];
          for (let i = 0; i < result.length; ++i) {
            this.tableItems.push({
              "订单号": result[i].prescriptionId,
              "时间": result[i].diagnoseTime.replace("T", " ").slice(0, 19),
              "就诊人ID": result[i].patientId,
              "金额": result[i].totalPrice,
              "状态": result[i].paystate ? "已支付" : "未支付",
            })
          }
          for (let i = 0; i < this.tableItems.length; ++i) {
            fetch("http://124.223.143.21/api/Patient/PatientDetails/" + this.tableItems[i]["就诊人ID"], {
              method: 'GET',
              redirect: 'follow'
            }).then(response => response.text())
              .then(result => {
                result = JSON.parse(result);
                this.tableItems[i]["就诊人姓名"] = result[0].name;
              })
              .catch(error => console.log('error', error));
          }
        })
        .catch(error => console.log('error', error));
    },

    // 点击表行
    openModal(event) {
      // 弹窗已弹出则操作无效
      if (this.showModal) {
        return;
      }

      // 从后端获取订单详细信息
      this.orderId = this.tableItems[event.itemIndex]["订单号"];
      this.selectedRow = event.itemIndex;
      fetch("http://124.223.143.21/api/Prescription/GetDetail?prescriptionId=" + this.orderId, {
        method: 'GET',
        redirect: 'follow'
      }).then(response => response.text())
        .then(result => {
          result = JSON.parse(result);
          this.modalItems = [];
          for (let i = 0; i < result.length; ++i) {
            this.modalItems.push({
              "药品名": result[i].medicineName,
              "数量": result[i].quantity,
              "单价": result[i].medicinePrice,
              "总价": result[i].quantity * result[i].medicinePrice,
            })
          }
          this.showModal = true; // 显示弹窗
        })
        .catch(error => console.log('error', error));
    },

    // 确认订单
    confirm() {
      /* 改变订单状态 */
      fetch("http://124.223.143.21/api/Prescription/UpdatePaystate"
        + "?prescriptionId=" + this.orderId.toString(), {
        method: 'PUT',
        redirect: 'follow'
      }).then(response => response.text())
        .then(result => {
          if (result == "该订单已支付") {
            alert(result);
            this.showModal = false; // 关闭弹窗
            return;
          }

          /* 获取所有药品 */
          fetch("http://124.223.143.21/api/Stock/GetAllStocks", {
            method: 'GET',
            redirect: 'follow'
          }).then(response => response.text())
            .then(result => {
              result = JSON.parse(result);
              for (let i in this.modalItems) {
                let flag = false;
                for (let j = 0; j < result.length; ++j) {
                  if (result[j].medicineName == this.modalItems[i]["药品名"]
                    && result[j].medicineAmount >= this.modalItems[i]["数量"]) {

                    /* 减少药品数量 */
                    fetch("http://124.223.143.21/api/Medicine/UpdateStock"
                      + "?medicineName=" + result[j].medicineName
                      + "&newAmount=" + (result[j].medicineAmount - this.modalItems[i]["数量"]).toString()
                      + "&manufacturer=" + result[j].manufacturer
                      + "&productionDate=" + result[j].productionDate
                      + "&administratorId=" + sessionStorage.getItem('userID')
                      + "&patientId=" + this.tableItems[this.selectedRow]["就诊人ID"], {
                      method: 'PUT',
                      redirect: 'follow'
                    }).then(response => response.text())
                      .then(result => console.assert(result == "Medicine stock and purchase record updated successfully."))
                      .catch(error => console.log('error', error));
                    /* 减少药品数量 */

                    flag = true;
                    break;
                  }
                }
                console.assert(flag);
              }
            })
            .catch(error => console.log('error', error));
          /* 获取所有药品 */

          this.getTable(); // 刷新表格
          this.showModal = false; // 关闭弹窗
        })
        .catch(error => console.log('error', error));
      /* 改变订单状态 */
    },

    // 取消操作
    cancel() {
      // 关闭弹窗
      this.showModal = false;
    }
  },

  mounted() {
    // 初始化表格和filteredCount
    this.getTable();
    this.filteredCount = this.tableItems.length;
  },

  watch: {
    // 重新筛选时重置curPage
    filter() {
      this.curPage = 1;
    }
  },
}
</script>

<style scoped>
.el-main {
  user-select: none;
}

* {
  /* 应用字体 */
  font-family: AliRegular;
  --va-font-family: AliRegular;
}

.va-data-table {
  --va-font-family: AliRegular;
  /* 表头大小 */
  --va-data-table-thead-font-size: 1rem;
}
</style>

