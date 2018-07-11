<template>
  <!-- developer: xuchao 2018.5.8 -->
  <div class="add-park">
    <div class="park-title">
      {{title}}
    </div>
    <el-form ref="form" :model="form" label-width="120px" class="park-form" :rules="rules">
      <el-collapse v-model="activeNames">
        <el-collapse-item class="base-info" title="基本信息" name="1">
          <el-form-item label="停车场名称：" prop="name">
            <el-input v-model="form.name" placeholder="请输入停车场名称"></el-input>
          </el-form-item>
          <el-form-item label="停车场类型：" prop="type">
            <el-select v-model="form.type" placeholder="选择停车场类型">
              <el-option key="0" label="场内停车场" value="0"></el-option>
              <el-option key="1" label="路边停车场" value="1"></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="停车场位置：" prop="addressDtl">
            <el-button class="btn-select-location btn-line-hover" type="primary" @click="selectLocation">{{form.longitude?'再次录入':'选择位置'}}</el-button>
            <div class="info-location">
              <span class="info-location-detail">位置：{{form.addressDtl}}</span>
              <span class="info-location-lng">经度：{{form.longitude}}</span>
              <span class="info-location-lat">纬度：{{form.latitude}}</span>
            </div>
          </el-form-item>
          <el-form-item label="开放时间：" prop= "openTime">
            <el-input v-model="form.openTime" placeholder="请输入停车场开放时间"></el-input>
          </el-form-item>
          <el-form-item label="其他属性：" prop="remark">
            <el-input v-model="form.remark" placeholder="填写内容"></el-input>
          </el-form-item>
        </el-collapse-item>
        <el-collapse-item class="park-fee" title="收费标准" name="2">
          <el-form-item label="收费摘要：" prop= "chargeBrief">
            <el-input v-model="form.chargeBrief" placeholder="建议输入不超过8字符"></el-input>
          </el-form-item>
          <el-form-item label="收费详情：" prop= "chargeDetail">
            <el-input type="textarea" v-model="form.chargeDetail" placeholder="输入具体的收费标准"></el-input>
          </el-form-item>
        </el-collapse-item>
        <el-collapse-item class="floor" title="楼层分组" name="3">
          <ul>
            <li v-for="(item, index) in form.groupList" v-bind:key="item.id" @click="getIndex(index)">
              <span class="delete-floor" v-if="index !== 0" @click="deleteFloor(index)"></span>
              <el-form-item label="楼层分组：" :prop="'groupList.'+index+'.groupName'" :rules="[{
                  required: true, message: '楼层分组不能为空，必填', trigger: 'blur'
                },{max:30,message:'最大长度为30个字符',trigger:['blur','change']}]" >
                <el-input v-model="item.groupName"></el-input>
              </el-form-item>
              <el-form-item label="本层分区：" :prop="'groupList.' +  index  + '.areaList'" :rules="[{max:30,message:'最大长度为30个字符',trigger:['blur','change']}]" >
                <el-input v-model="item.areaList" placeholder="输入多个分区之间用“；”分开，不填写则默认为一个区"></el-input>
              </el-form-item>
              <el-form-item label="添加车位图：" :prop="'groupList.'+index+'.fileList'" :rules="{required: true, message: '车位图不能为空，必填', trigger: 'blur,change'}" :ref="'uploadSvg'+index">
                <el-upload class="upload-demo" action="/park/park-parkingloting/parkingFloor/webInfo/uploadFloorMap" :multiple="false" accept='.svg' :headers="uploadHeaders"
                  :auto-upload="true" :limit="1" :on-exceed="handleExceed" :before-upload="checkFileType(index,item.fileList[0])" :on-success="uploadSuc" :on-error="uploadErr" :before-remove="removeFile"
                  :file-list="item.fileList" :on-change="svgChange" :on-remove="onRemove">
                  <el-button class="btn-select-map btn-line-hover" size="small" type="primary">选择车位图文件</el-button>
                   <span slot="tip" class="el-upload__tip">只能上传svg文件，且不超过500kb</span>
                </el-upload>
              </el-form-item>
            </li>
          </ul>
          <el-button class="add-park-map" type="primary" @click="addFloor">
            <span class="btn-plus">+</span>
            <span class="btn-words">添加楼层分组</span>
          </el-button>
        </el-collapse-item>
      </el-collapse>
    </el-form>
    <div class="btn-submit">
      <el-button class="add-park-map" @click="cancel">取 消</el-button>
      <el-button class="add-park-map" type="primary" @click="submitForm('form')">确 定</el-button>
    </div>
    <mapDrag class="map" v-if="showMap" v-on:returnPosition="getPositon" :form-info="form"></mapDrag>
    <div class="modal" v-if="showModal"></div>
  </div>
</template>

<script>
  import mapDrag from "@/modules/mapDrag.vue";
  import {errorService} from "../utils";
  export default {
    name: "addPark",
    components: {
      mapDrag
    },
    data() {
      return {
        title: "新增停车场",
        uploadHeaders:{},
        parkId: "",
        activeNames: ["1", "2", "3"],
        form: {
          name: "",
          type: "",
          addressDtl: "",
          longitude: "",
          latitude: "",
          openTime: "",
          remark: "",
          chargeBrief: "",
          chargeDetail: "",
          groupList: [{
            groupName: "",
            areaList: "",
            fileList: []
          }],
        },
        uploadIndex: 0,
        rules: {
          name: [{
              required: true,
              message: '停车场名称不能为空，必填',
              trigger: 'blur'
            },
            {
              min: 1,
              max: 30,
              message: '最大长度为30个字符',
              trigger: 'blur'
            }
          ],
          type: [{
            required: true,
            message: '停车场类型不能为空，必选',
            trigger: 'blur'
          }],
          addressDtl: [{
            required: true,
            message: '停车场位置不能为空，必选',
            trigger: 'blur'
          }],
          remark:[
            {
                max:500,
                message:'最大长度为500个字符',
                trigger:['blur','change']
            }
          ],
          openTime:[
            {
              max:30,
              message:'最大长度为30个字符',
              trigger:['blur','change']
            }
          ],
          chargeBrief:[
            {
              max:30,
              message:'最大长度为30个字符',
              trigger:['blur','change']
            }
          ],
          areaList:[
            {
              max:30,
              message:'最大长度为30个字符',
              trigger:['blur','change']
            }
          ]

        },
        parkMap: [],
        showModal: false,
        showMap: false,
        pageFlag: this.$route.params.pageFlag,
        userId: sessionStorage.getItem("userId"),
        createMgrId:'',
        deleteGroupList: []
      };
    },
    created() {
      this.createMgrId = JSON.parse(
            localStorage.getItem("_CMIOT_SmartPark_userInfo_" + this.userId)
          ).mgrId
      if(localStorage.getItem("_CMIOT_SmartPark_userInfo_" + this.userId)){
        this.uploadHeaders={
          token:JSON.parse(
            localStorage.getItem("_CMIOT_SmartPark_userInfo_" + this.userId)
          ).token,
          auth:'smartPark'
        }
      }else{
        this.$message.warning('未能获取到用户信息！');
      }
      if (this.pageFlag == 1) {
        this.title = "编辑停车场";
        this.parkId = JSON.parse(
          localStorage.getItem("_CMIOT_SmartPark_parkId_" + this.userId)
        ).parkId;
        if (this.parkId) {
          this.searchPark(this.parkId);
        } else {
          this.$message.warning('未能获取到停车场ID！');
        }
      }
    },
    methods: {
      searchPark(parkId) {
        this.$ajax.get('/park-parkingloting/parkingLot/webInfo/get/' + parkId).then(res => {
          if (res && res.data) {

            var resData = res.data.data;
            this.form = resData;
            this.form.type = String(this.form.type);
          } else {
            this.$message.warning('查询的停车场信息不正确！');
          }
        }).catch(err => {
          errorService(this,err);
        });
      },
      selectLocation() {
        this.showMap = true;
        this.showModal = true;
      },
      getPositon(e) {
        this.showMap = false;
        this.showModal = false;

        if (e) {
          this.form.addressDtl = e.address;
          this.form.longitude = e.position.lng;
          this.form.latitude = e.position.lat;
        }
      },
      addFloor() {
        this.form.groupList.push({
          name: "",
          areaList: "",
          fileList: []
        });
      },
      deleteFloor(index) {
        if (this.form.groupList[index].fileList.length > 0) {
          this.$confirm('删除楼层则车位图也会删除，是否确认删除?', '', {
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
          }).then(() => {
            var data = {
              "id": new Date().getTime(),
              "data": {
                "fileId": this.form.groupList[index].fileList[0].response.data,
                "floorGroupId":this.form.groupList[index].floorGroupId?this.form.groupList[index].floorGroupId:''
              }
            }
            var headers = {
              'Content-Type': 'application/json;charset=UTF-8'
            };
            this.deleteFile(data, headers);
            if (this.form.groupList[index].floorGroupId) {
              this.deleteGroupList.push(this.form.groupList[index].floorGroupId);
            }
            this.form.groupList.splice(index, 1);
          })
        } else {
          if (this.form.groupList[index].floorGroupId) {
            this.deleteGroupList.push(this.form.groupList[index].floorGroupId);
          }
          this.form.groupList.splice(index, 1);
        }
      },
      svgChange(file) {
        if(file && file.status === 'success') {
          this.$refs['uploadSvg'+this.uploadIndex][0].clearValidate()
        }
      },
      handleExceed() {
        this.$message.warning('文件上传超过限制,一个楼层仅可上传一张车位图！');
      },
      checkFileType(index,file) {
        if(file && file.name) {
          if(file.name.indexOf('.svg') === -1) {
            this.$message.warning('只能上传svg文件');
            this.form.groupList[index].fileList = [];
            return false;
          }
        }
        if( file && file.size && file.size > 500*1024) {
          this.$message.warning('文件大小超出限制');
          this.form.groupList[index].fileList = [];
          return false;
        }
      },
      uploadSuc(response, file) {
        if (response.code === 10000) {
            this.$message.success('上传车位图成功！');
          } else {
            this.$message.warning('上传车位图失败！');
          }
        this.form.groupList[this.uploadIndex].fileList = [];
        this.form.groupList[this.uploadIndex].fileList.push(file);
      },
      uploadErr(err) {
        if(err.status === 401) {
          this.$message.error("登录超时，请重新登录");
          this.$router.replace('/login');
          return;
        }
        this.$message.warning(`车位图上传失败，请联系管理员！`);
      },
      removeFile(file, fileList) {
        if (!file.response) {
          return false;
        }
        this.$confirm('是否确认删除?', '', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          var index = false;
          for (var i = 0, len = this.form.groupList.length; i < len; i++) {
            if (this.form.groupList[i].fileList[0].response.data === file.response.data) {
              index = i;
            }
          }
          if (index || index === 0) {
            var floorGroupId = this.form.groupList[index].floorGroupId;
          }
          var data = {
            "id": new Date().getTime(),
            "data": {
              "fileId": file.response.data,
              "floorGroupId":floorGroupId?floorGroupId:''
            }
          }
          var headers = {
            'Content-Type': 'application/json;charset=UTF-8'
          };
          this.deletePath(data);
          this.deleteFile(data, headers, fileList, file);
          return true;
        }).catch(()=>{})
      },
      onRemove(file,fileList) {
        fileList.push(file);
      },
      deleteFile(data, headers, fileList, file) {

        this.$ajax.delete('/park-parkingloting/parkingFloor/webInfo/deleteFloorMap', data, headers).then(res => {

          if (res.data.code === 10000) {

            this.$message.success('删除车位图成功');
          } else {
            this.$message.warning('删除车位图失败');
            if (fileList && fileList.length === 1) {
              fileList.push(file);
            }
          }
        }).catch(err => {
          errorService(this,err);
          if (fileList && fileList.length === 1) {
            fileList.push(file);
          }
        });
        return;
      },
      deletePath(data) {
        var index = false;
        for (var i = 0, len = this.form.groupList.length; i < len; i++) {
          if (this.form.groupList[i].fileList[0].response.data === data.data.fileId) {
            index = i;
          }
        }
        if (index || index === 0) {
          this.form.groupList[index].fileList = [];
        }
        return;
      },
      getIndex(index) {
        this.uploadIndex = index;
      },
      submitForm(formName) {
        this.$refs[formName].validate((valid) => {
          if (valid) {
            var address = this.form.addressDtl.substr(0, this.form.addressDtl.indexOf('区') + 1);
            var params = {
              "id": new Date().getTime(),
              "data": {
                "parkId": this.form.parkId,
                "name": this.form.name,
                "type": this.form.type,
                "address": address,
                "addressDtl": this.form.addressDtl,
                "longitude": this.form.longitude,
                "latitude": this.form.latitude,
                "chargeBrief": this.form.chargeBrief,
                "chargeDetail": this.form.chargeDetail,
                "chargePolicy": "",
                "openTime": this.form.openTime,
                "remark": this.form.remark,
                "createMgrId": this.createMgrId,
                "groupList": this.form.groupList
              }
            }
            var groupList = params.data.groupList.map(function (val) {
              return val.groupName;
            })
            var arr = groupList.sort();
            for(var i=0; i< groupList.length; i++) {
              if (arr[i] === arr[i+1]) {
                this.$message.warning('楼层分组'+arr[i]+'重复');
                return false;
              }
            }
            if (this.pageFlag == 1) {
              this.editPark(params)
            } else {
              this.addPark(params)
            }

          } else {
            return false;
          }
        });
      },
      addPark(params) {
        this.$ajax.post('/park-parkingloting/parkingLot/webInfo/insert', params).then(res => {
          if (res && res.data.code === 10000) {
            if(res.data.data && res.data.data[0].responseCode === 10615) {
              this.$message.warning('楼层分组'+res.data.data[0].groupName+'重复');
              return false;
            }
            this.$message.success('新增停车场成功！');
            this.cancel();
          } else if(res.data.code === 10608) {
            this.$message.warning('停车场名称已存在，新增停车场失败');
          } else {
            this.$message.warning('新增停车场失败！');
          }
        }).catch(err => {
          errorService(this,err);
        });
      },
      editPark(params) {
        params.data.deleteGroupList = this.deleteGroupList;
        this.$ajax.put('/park-parkingloting/parkingLot/webInfo/update', params).then(res => {
          if (res && res.data.code === 10000) {
            if(res.data.data && res.data.data[0].responseCode === 10615) {
              this.$message.warning('楼层分组'+res.data.data[0].groupName+'重复');
              return false;
            }
            this.$message.success('编辑停车场成功！');
            this.cancel();
          } else {
            this.$message.warning('编辑停车场失败！');
          }
        }).catch(err => {
          errorService(this,err);
        });
      },
      cancel() {
        this.$router.push({
          path: "/managePark"
        });
      }
    }
  };

</script>
<style lang="scss">
  .main {
    .add-park {
      min-width: 900px;
      .park-title {
        margin-bottom: 1px;
        height: 60px;
        background: #fff;
        font-family: "MicrosoftYaHei";
        font-size: 20px;
        color: #353b38;
        line-height: 60px;
        text-indent: 20px;
      }
      .park-form {
        background: #fff;
        padding: 20px 60px 0;
        .el-collapse {
          border-top: none;
        }
        .el-collapse-item {
          .el-collapse-item__header {
            height: 60px;
            font-family: "MicrosoftYaHei";
            font-size: 18px;
            color: #353b38;
            line-height: 60px;
            border-bottom: 1px dashed rgba(0, 0, 0, 0.15);
            .el-collapse-item__arrow {
              position: relative;
              top: 8px;
            }
          }
          .el-collapse-item__wrap {
            border-bottom: none;
            .el-collapse-item__content {
              padding: 30px 0 30px 210px;
              font-family: "MicrosoftYaHei";
              font-size: 14px;
              color: #353b38;
              .upload-demo {
                height: 40px;
              }
              .el-form-item {
                margin-bottom: 16px;
              }
              .el-input__inner {
                width: 400px;
                height: 32px;
                border-radius: 16px;
              }
              .el-textarea__inner {
                width: 400px;
                height: 120px;
                background: #ffffff;
                border: 1px solid rgba(0, 0, 0, 0.15);
                border-radius: 16px;
                resize: none;
              }
              .btn-select-location {
                width: 100px;
                height: 32px;
                border-radius: 16px;
                line-height: 8px;
                font-size: 14px;
                background: #ffffff;
                color: #52b98d;
                border: 1px solid #52b98d;
              }
              .btn-select-map {
                width: 140px;
                height: 32px;
                border-radius: 16px;
                line-height: 8px;
                font-size: 14px;
                background: #ffffff;
                color: #52b98d;
                border: 1px solid #52b98d;
              }
              .el-upload__tip {
                margin-left: 10px;
              }
              .el-upload-list {
                position: relative;
                top: 12px;
                left: 0;
                width: 351px;
                .el-upload-list__item {
                  margin: 0;
                  padding: 0;
                  border: none;
                  background: transparent;
                  height: 30px;
                  width: 255px;
                }
              }
              .info-location {
                position: relative;
                top: -21px;
                display: inline-block;
                width: 573px;
                line-height: 32px;
                margin-left: 10px;
                .info-location-detail {
                  position: absolute;
                  top: -10px;
                  font-size: 12px;
                }
                .info-location-lng {
                  position: absolute;
                  top: 15px;
                  left: 0px;
                  display: inline-block;
                  font-size: 12px;
                  width: 105px;
                  line-height: 16px;
                }
                .info-location-lat {
                  position: absolute;
                  top: 15px;
                  left: 107px;
                  display: inline-block;
                  width: 105px;
                  font-size: 12px;
                  line-height: 16px;
                }
              }
              ul {
                margin: 0;
                padding: 0;
                li {
                  position: relative;
                  list-style: none;
                  padding-top: 16px;
                  width: 630px;
                  height: 200px;
                  background: #fafafa;
                  border: 1px solid rgba(0, 0, 0, 0.05);
                  margin-bottom: 20px;
                  .delete-floor {
                    position: absolute;
                    left: 600px;
                    top: 8px;
                    display: inline-block;
                    width: 23px;
                    height: 23px;
                    background-image: url("/static/img/spirit.png");
                    background-repeat: no-repeat;
                    background-position: -331px -32px;
                    cursor: pointer;
                    z-index: 10;
                  }
                }
              }
            }
          }
        }
      }
      .floor {
        .add-park-map {
          width: 630px;
          height: 32px;
          border-radius: 16px;
          line-height: 8px;
          font-size: 14px;
          background: #ffffff;
          color: #52b98d;
          border: 1px dashed #52b98d;
          .btn-plus {
            position: relative;
            top: -1px;
            font-size: 24px;
            font-weight: 700;
          }
          .btn-words {
            position: relative;
            top: -4px;
          }
        }
      }
      .btn-submit {
        height: 60px;
        text-align: center;
        line-height: 60px;
        margin-bottom: 83px;
        font-size: 0px;
        button {
          width: 100px;
          height: 32px;
          line-height: 8px;
          font-size: 14px;
          margin-right: 10px;
          border-radius: 16px;
        }
      }
      .modal {
        position: fixed;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        z-index: 999;
        background: rgba(0, 0, 0, 0.45);
      }
      .map {
        position: fixed;
        width: 960px;
        height: 540px;
        top: 50%;
        left: 50%;
        border: 1px solid rgba(0, 0, 0, 0.15);
        z-index: 9999;
        transform: translate(-480px, -270px);
      }
    }
  }

</style>
