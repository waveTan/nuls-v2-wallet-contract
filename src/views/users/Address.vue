<template>
  <div class="address bg-gray">
    <h3 class="title">{{$t('address.address0')}}</h3>

    <div class="w1200 mt_20">
      <div class="top_ico">
        <i class="el-icon-plus click" @click="toUrl('newAddress')"></i>
      </div>
      <el-table :data="addressList" stripe border>
        <el-table-column prop="address" :label="$t('address.address1')" align="center" min-width="200">
        </el-table-column>
        <el-table-column prop="balance" :label="$t('address.address2')" align="center">
        </el-table-column>
        <el-table-column :label="$t('address.address3')" align="center">
          <template slot-scope="scope">
            <span v-show="scope.row.alias">{{scope.row.alias}}</span>
            <span v-show="!scope.row.alias" @click="addAlias(scope.row)">
              <i class="el-icon-edit-outline click"></i>
            </span>
          </template>
        </el-table-column>
        <el-table-column :label="$t('address.address5')" align="center" width="350">
          <template slot-scope="scope">
            <!-- <label class="click tab_bn" @click="editPassword(scope.row)">{{$t('address.address6')}}</label>
             <span class="tab_line">|</span>-->
            <label class="click tab_bn" @click="backAddress(scope.row)">{{$t('address.address7')}}</label>
            <span class="tab_line">|</span>
            <label class="click tab_bn" @click="deleteAddress(scope.row)">{{$t('address.address8')}}</label>
            <span class="tab_line">|</span>
            <el-link disabled v-if="scope.row.selection">{{$t('public.into')}}</el-link>
            <label class="click tab_bn" @click="selectionAddress(scope.row)" v-else>{{$t('public.into')}}</label>
          </template>
        </el-table-column>
      </el-table>
      <div class="pages">
        <div class="page-total">{{$t('public.total')}} {{addressList.length}}</div>
      </div>
    </div>

    <el-dialog title="Remarks" width="30rem"
               :visible.sync="remarkDialog"
               :close-on-click-modal="false"
               :close-on-press-escape="false"
    >

      <div class="address-remark bg-white">
        <el-input v-model.trim="remarkInfo" :placeholder="$t('address.address9')"></el-input>
        <div class="btn-next">
          <el-button @click="remarkDialog=false">{{$t('address.address10')}}</el-button>
          <el-button type="success" @click='addRemark'>{{$t('address.address11')}}</el-button>
        </div>
      </div>
    </el-dialog>

    <Password ref="password" @passwordSubmit="passSubmit">
    </Password>
  </div>
</template>

<script>
  import nuls from 'nuls-sdk-js'
  import Password from '@/components/PasswordBar'
  import {timesDecimals, chainIdNumber, addressInfo, chainID} from '@/api/util'
  import {LOCALHOST_API_URL, PARAMETER} from '@/config.js'
  import axios from 'axios'

  export default {
    data() {
      return {
        addressList: [],//地址列表
        selectAddressInfo: '', //操作的地址信息
        remarkDialog: false,//备注弹框
        remarkInfo: '',//备注信息
      };
    },
    components: {
      Password,
    },
    created() {
      this.getAddressList();
    },
    mounted() {
      this.getAddressLists(this.addressList);
    },
    methods: {

      /**
       * 获取账户列表
       */
      async getAddressList() {
        this.addressList = await addressInfo();
        //如果没有账户跳转到创建地址界面
        if (this.addressList.length === 0) {
          this.$router.push({
            name: "newAddress",
            query: {'address': ''}
          })
        }
      },

      /**
       * 获取地址网络信息
       * @param addressInfo
       **/
      async getAddressInfoByNode(addressInfo) {
        await this.$post('/', 'getAccount', [addressInfo.address])
          .then((response) => {
            //console.log(response);
            if (response.hasOwnProperty("result")) {
              for (let item of this.addressList) {
                if (item.address === addressInfo.address) {
                  addressInfo.alias = response.result.alias;
                  addressInfo.balance = timesDecimals(response.result.balance);
                  addressInfo.consensusLock = timesDecimals(response.result.consensusLock);
                  addressInfo.totalReward = timesDecimals(response.result.totalReward);
                  addressInfo.tokens = [];
                  addressInfo.chainId = nuls.verifyAddress(item.address).chainId;
                }
              }
              //localStorage.setItem(chainIdNumber(), JSON.stringify(this.addressList))
            }
          })
          .catch((error) => {
            console.log("getAccount:" + error);
          });
      },

      /**
       * 循环获取账户余额及别名
       * @param addressList
       **/
      getAddressLists(addressList) {
        for (let item of addressList) {
          setTimeout(() => {
            this.getAddressInfoByNode(item);
          }, 500);
        }
      },

      /**
       * 设置别名
       * @param rowInfo
       **/
      addAlias(rowInfo) {
        if (rowInfo.balance === 0) {
          this.$message({message: this.$t('address.address12'), type: 'error', duration: 1000});
        } else {
          this.toUrl('setAlias', rowInfo.address)
        }
      },

      /**
       *  修改密码
       * @param rowInfo
       **/
      editPassword(rowInfo) {
        this.toUrl('editPassword', rowInfo.address)
      },

      /**
       *  备份账户
       * @param rowInfo
       **/
      backAddress(rowInfo) {
        this.selectAddressInfo = rowInfo;
        this.$router.push({
          name: "newAddress",
          query: {'backAddressInfo': rowInfo}
        })
      },

      /**
       *  移除账户
       * @param rowInfo
       **/
      deleteAddress(rowInfo) {
        this.selectAddressInfo = rowInfo;
        this.$refs.password.showPassword(true)
      },

      /**
       * 进入账户（使用账户）
       * @param rowInfo
       **/
      selectionAddress(rowInfo) {
        //console.log(rowInfo);
        for (let item  of this.addressList) {
          //清除选中
          if (item.selection) {
            item.selection = false;
          }
          //添加选中
          if (item.address === rowInfo.address) {
            item.selection = true;
          }
        }
        localStorage.setItem(chainIdNumber(), JSON.stringify(this.addressList));
        this.$router.push({
          name: 'home',
        })
      },

      /**
       *  获取密码框的密码
       * @param password
       **/
      passSubmit(password) {
        PARAMETER.method = 'deleteAccount';
        PARAMETER.params = [chainID(), this.selectAddressInfo.address, password];
        axios.post(LOCALHOST_API_URL, PARAMETER)
          .then((response) => {
            //console.log(response.data);
            if (response.data.hasOwnProperty('result')) {
              this.getAddressList()
            }
          }).catch((err) => {
          console.log(err)
        });
      },

      /**
       *  编辑账户备注弹框
       * @param rowInfo
       */
      editRemark(rowInfo) {
        this.selectAddressInfo = rowInfo;
        this.remarkInfo = this.selectAddressInfo.remark;
        this.remarkDialog = true
      },

      /**
       * 账户备注提交
       */
      addRemark() {
        let newAddressInfo = addressInfo(0);
        for (let item of newAddressInfo) {
          if (item.address === this.selectAddressInfo.address) {
            this.selectAddressInfo.remark = this.remarkInfo;
            item.remark = this.remarkInfo;
          }
        }
        localStorage.setItem(chainIdNumber(), JSON.stringify(newAddressInfo));
        this.remarkDialog = false;
        this.selectAddressInfo = '';
      },

      /**
       * 连接跳转
       * @param name
       * @param param
       */
      toUrl(name, param) {
        //console.log(name)
        this.$router.push({
          name: name,
          query: {'address': param}
        })
      },
    }
  }
</script>

<style lang="less">
  @import "./../../assets/css/style";

  .address {
    .el-dialog__body {
      background-color: @Bcolour;
      padding: 30px 20px 50px 20px;
      .address-remark {
        margin: 50px auto 0;
        .btn-next {
          margin: 40px auto 0;
          text-align: center;
          .el-button {
            width: 9.5rem;
          }
          .el-button--success {
            span {
              color: white;
            }
          }
        }
      }
    }
  }


</style>
