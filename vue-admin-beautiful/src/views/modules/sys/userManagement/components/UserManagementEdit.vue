<template>
  <el-dialog
    :title="title"
    :visible.sync="dialogFormVisible"
    width="800px"
    @close="close"
  >
    <el-form ref="form" :model="form" :rules="rules" label-width="105px" class="userManagement-edit-container">

      <el-row>
        <el-col :span="12">
          <!-- 大型项目设计都不是不允许修改用户名的 可能会关联很多地方 -->
          <!-- 修改不允许改用户名 -->
          <el-form-item label="用户名" prop="username">
            <el-input v-model.trim="form.username" autocomplete="off"
                      :disabled="!formStatus"
            ></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="昵称" prop="realName">
            <el-input v-model="form.realName" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
      </el-row>

      <!-- 如果是新增 则可以直接设置用户密码 修改则不允许 -->
      <el-row v-if="formStatus">
        <el-col :span="12">
          <el-form-item label="密码" prop="password">
            <el-input
                :type="formPasswordShow.pass?'text':'password'"
                v-model.trim="form.password"
                autocomplete="off"
            ></el-input>
            <span class="show-password" @click="formPasswordShow.pass = !formPasswordShow.pass">
              <vab-icon v-if="formPasswordShow.pass" :icon="['fas', 'eye-slash']"></vab-icon>
              <vab-icon v-else :icon="['fas', 'eye']"></vab-icon>
            </span>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="确认密码" prop="verifyPassword">
            <el-input
              :type="formPasswordShow.verify?'text':'password'"
              v-model.trim="form.verifyPassword"
              autocomplete="off"
            ></el-input>
            <span class="show-password" @click="formPasswordShow.verify = !formPasswordShow.verify">
              <vab-icon v-if="formPasswordShow.verify" :icon="['fas', 'eye-slash']"></vab-icon>
              <vab-icon v-else :icon="['fas', 'eye']"></vab-icon>
            </span>
          </el-form-item>
        </el-col>
      </el-row>

      <el-row>
        <el-col :span="12">
          <el-form-item label="工号" prop="no">
            <el-input v-model.trim="form.no" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="手机" prop="mobile">
            <el-input v-model.trim="form.mobile" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
      </el-row>

      <el-row>
        <el-col :span="12">
          <el-form-item label="邮箱" prop="email">
            <el-input v-model.trim="form.email" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="是否锁定" prop="izLock">
            <el-select v-model="form.locked" placeholder="请选择" style="width: 100%">
              <el-option
                v-for="item in dict.no_yes"
                :key="item.dictValue"
                :label="item.dictName"
                :value="item.dictValue"
              ></el-option>
            </el-select>
          </el-form-item>
        </el-col>
      </el-row>

      <el-row>
        <el-col :span="12">
          <el-form-item label="签名" prop="sign">
            <el-input type="textarea" v-model="form.sign" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
        <el-col :span="12">
          <el-form-item label="备注" prop="remark">
            <el-input type="textarea" v-model="form.remark" autocomplete="off"></el-input>
          </el-form-item>
        </el-col>
      </el-row>

      <!-- 如果是超级管理员 可以设置租户 -->
      <el-row v-if="userInfo != null && userInfo.izSuperAdmin" >
        <el-col :span="12">
          <el-form-item label="租户ID" prop="icon">
            <el-input v-model="form.tenantId" autocomplete="off" readonly ></el-input>
            <el-button type="primary" icon="el-icon-search"
                       class="input-btn-choose" @click="showTenant"></el-button>
          </el-form-item>
        </el-col>
      </el-row>

    </el-form>
    <div slot="footer" class="dialog-footer">
      <el-button @click="close">取 消</el-button>
      <el-button type="primary" @click="save">确 定</el-button>
    </div>

    <tenant
      v-on:tenant="closeTenant"
      ref="tenant"
    ></tenant>

  </el-dialog>
</template>

<script>
  import { doInsert, doUpdate } from "@/api/userManagement";
  import { getAccessToken } from "@/utils/accessToken";
  import { getUserInfo } from "@/api/user";
  import Tenant from "@/components/opsli/tenant/tenant";
  import {isCode, isPhone, isName, isNull, isPassword, isEmail} from "@/utils/validate";

  export default {
    name: "UserManagementEdit",
    components: { Tenant },
    data() {

      const validateUsername = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error("用户名不能为空"));
        } else if (!isCode(value)) {
          callback(new Error('用户名只能为字母、数字或下划线'));
        } else {
          callback();
        }
      };

      const validatePassword = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error('请输入密码'));
        } else if (!isPassword(value)) {
          callback(new Error("密码至少包含大小写字母，数字，且不少于6位"));
        } else {
          callback();
        }
      };

      const validateVerifyPassword = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error('请再次输入密码'));
        } else if (value !== this.form.password) {
          callback(new Error('两次输入密码不一致'));
        } else {
          callback();
        }
      };

      const validateRealName = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error("请输入昵称"));
        } else if (!isName(value)) {
          callback(new Error("请输入正确的昵称格式"));
        } else {
          callback();
        }
      };

      const validateNo = (rule, value, callback) => {
        if (isNull(value)) {
          callback(new Error('请输入工号'));
        } else if (!isCode(value)) {
          callback(new Error('工号只能为字母、数字或下划线'));
        } else {
          callback();
        }
      };

      const validateMobile = (rule, value, callback) => {
        if (!isNull(value) && !isPhone(value)) {
          callback(new Error('请输入正确的手机号格式'));
        } else {
          callback();
        }
      };

      const validateEmail = (rule, value, callback) => {
        if (!isNull(value) && !isEmail(value)) {
          callback(new Error('请输入正确的邮箱格式'));
        } else {
          callback();
        }
      };

      return {
        userInfo: null,
        dict: {},
        formStatus: true,
        formPasswordShow: {
          pass: false,
          verify: false,
        },
        form: {
          tenantId:"",
          locked: '0',
          // 设置默认值
          version: 0
        },
        rules: {
          username: [
            { required: true, trigger: "blur", validator: validateUsername },
          ],
          password: [
            { required: true, trigger: "blur", validator: validatePassword },
          ],
          verifyPassword: [
            { required: true, trigger: "blur", validator: validateVerifyPassword },
          ],
          realName: [
            { required: true, trigger: "blur", validator: validateRealName },
          ],
          no: [
            { required: true, trigger: "blur", validator: validateNo },
          ],
          mobile: [
            { required: false, trigger: "blur", validator: validateMobile },
          ],
          email: [
            { required: false, trigger: "blur", validator: validateEmail },
          ],
        },
        title: "",
        dialogFormVisible: false,
      };
    },
    created() {
      this.getUser();
    },
    mounted() {
      // 如果不是每次开启时查询 在created中 有可能会短暂查不到
      this.dict.no_yes =  this.$getDictList("no_yes")
    },
    methods: {
      // 展示租户
      showTenant(){
        this.$refs["tenant"].show();
      },
      // 租户关闭
      closeTenant(val){
        this.form.tenantId = val.id;
      },
      showEdit(row) {
        if (!row) {
          this.title = "添加";
        } else {
          this.title = "编辑";
          this.formStatus = false;
          this.form = Object.assign({}, row);
        }
        this.dialogFormVisible = true;
      },
      close() {
        this.dialogFormVisible = false;
        this.$refs["form"].resetFields();
        this.formStatus = true;
        this.form = this.$options.data().form;
        this.$emit("fetchData");
      },
      save() {
        this.$refs["form"].validate(async (valid) => {
          if (valid) {
            // 修改
            if (!isNull(this.form.id)) {
              const { success, msg } = await doUpdate(this.form);
              if(success){
                this.$baseMessage(msg, "success");
              }
            } else {
              const { success, msg } = await doInsert(this.form);
              if(success){
                this.$baseMessage(msg, "success");
              }
            }

            await this.$emit("fetchData");
            this.close();
          } else {
            return false;
          }
        });
      },
      // 获取当前登录用户数据
      async getUser() {
        this.listLoading = true;
        let accessToken = getAccessToken();
        const { data } = await getUserInfo(accessToken);
        if(!isNull(data)){
          this.userInfo = Object.assign({}, data);
          setTimeout(() => {
            this.listLoading = false;
          }, 300);
        }
      },
    },
  };
</script>
<style lang="scss" scoped>
.userManagement-edit-container {

  .show-password {
    position: absolute;
    top: 1px;
    right: 13px;
    font-size: 16px;
    color: #d7dee3;
    cursor: pointer;
    user-select: none;
  }

}
</style>
