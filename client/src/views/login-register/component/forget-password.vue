<template>
  <el-form :model="form" :rules="rules" ref="formRef" class="login-form">
    <el-form-item prop="username">
      <el-input v-model="form.username" :prefix-icon="'User'" placeholder="邮箱账号" size="large">
        <template #append>
          <el-select v-model="form.username_suffix" placeholder="请选择邮箱类型" style="width: 120px" size="large">
            <email-options></email-options>
          </el-select>
        </template>
      </el-input>
    </el-form-item>
    <el-form-item class="verify-code" prop="verify_code">
      <el-input v-model="form.verify_code" placeholder="请输入图形验证码" size="large"></el-input>
      <img :src="imgCode.code" title="看不清，点击图片换一张" alt="" @click="getImgCode">
    </el-form-item>
    <el-form-item class="email-code" prop="sms_code">
      <el-input v-model="form.sms_code" placeholder="请输入邮箱验证码" size="large">
        <template #suffix>
          <el-button text type="primary" @click="getSmsCode" v-loading="sms_loading" :disabled="imgCode.counter !== 60">
            {{ imgCode.msg }}
          </el-button>
        </template>
      </el-input>
    </el-form-item>
    <el-form-item prop="password">
      <el-input v-model="form.password" :prefix-icon="'Lock'" show-password placeholder="设置新密码(8-20位字母、数字或字符，至少包含两种)" size="large"></el-input>
    </el-form-item>
    <el-form-item prop="rpassword" style="margin-bottom: 10px;">
      <el-input v-model="form.rpassword" :prefix-icon="'Lock'" show-password placeholder="确认新密码" size="large"></el-input>
    </el-form-item>
    <el-form-item prop="remember" class="register-enter">
      <span></span>
      <span>返回，去 <router-link :to="'/login'">登录</router-link></span>
    </el-form-item>
    <el-form-item class="login_btn">
      <el-button :loading="loading" type="primary" size="large" @click.native="forget">确定重置</el-button>
    </el-form-item>
  </el-form>
</template>

<script lang="ts" setup>
import { useCtxInstance, useMd5 } from '@/hooks/global';
import Users from '@/types/User';
import { VerifyCodeInter } from '@/typings/interface'
import VerifyCode from '@/types/VerifyCode';
import { computed, reactive, Ref, ref } from 'vue';
import { useRouter } from 'vue-router';
import { FormInstance } from 'element-plus';
import EmailOptions from './email-options.vue'

/**
 * 实例
 */
const user = new Users()
const verifyCode = new VerifyCode()
const router = useRouter()
const ctx = useCtxInstance()


/**
 * 变量
 */
const loading = ref(false)
const sms_loading = ref(false)
const form = reactive({
  username: '',
  username_suffix: '@163.com',
  password: '',
  rpassword: '',
  verify_code: '',
  sms_code: '',
  verify_id: '',
  remember: true // 记住密码
})
const formRef: Ref<FormInstance | null> = ref()
const rules = reactive({
  username: [
    { required: true, message: '请输入邮箱', trigger: ['blur'] },
    // {
    //   validator: (rule, value, callback) => {
    //     if (!/^\w{3,}(\.\w+)*@[A-z0-9]+(\.[A-z]{2,5}){1,2}$/.test(value)) {
    //       return callback('邮箱格式不正确')
    //     }
    //     callback()
    //   }
    // }
  ],
  password: [
    { required: true, message: '请输入密码', trigger: ['blur'] },
    {
      trigger: ['blur', 'change'],
      validator: (rule, value, callback) => {
        if (!/^(?!.*\s)(?!^[\u4e00-\u9fa5]+$)(?!^[0-9]+$)(?!^[A-z]+$)(?!^[^A-z0-9]+$)^.{8,16}$/.test(value)) {
          callback(new Error('8~20位字母、数字或字符，至少包含两种'))
        } else {
          callback()
        }
      }
    }
  ],
  rpassword: [
    { required: true, message: '请确认密码', trigger: ['blur'] },
    {
      trigger: ['blur', 'change'],
      validator: (rule, value, callback) => {
        if (value.trim() === '') {
          callback(new Error('请再次输入密码'))
        } else if (value !== form.password) {
          callback(new Error('两次输入密码不一致!'));
        } else {
          callback()
        }
      }
    }
  ],
  verify_code: [
    { required: true, message: '请输入图形验证码', trigger: ['blur'] }
  ],
  sms_code: [
    { required: true, message: '请输入邮箱验证码', trigger: ['blur'] }
  ]
})
// 图形验证码
const imgCode = reactive({
  id: '-',
  code: '',
  msg: '获取验证码',
  counter: 60
})
// 计算出实际邮箱地址
const email = computed(() => {
  return form.username + form.username_suffix
})


/**
 * 回调函数
 */
const forget = () => {
  formRef.value.validate(valid => {
    if (valid) {
      user.forget({
        account: email.value,
        password: useMd5(form.password),
        sms_code: form.sms_code
      }).then((res: any) => {
        ctx.$message({ message: '密码重置成功，去登录', type: 'success', duration: 1000 })
        router.push({ path: '/login' })
      }).catch(error => {
        ctx.$message({ message: error.message, type: 'error', duration: 1000 })
        if (error.message !== '验证码不正确') {
          getImgCode()
        }
        loading.value = false
      })
    }
  })
}
// 获取图形验证码
const getImgCode = () => {
  verifyCode.create({
    last_id: imgCode.id ? imgCode.id : '-'
  }).then((res: VerifyCodeInter) => {
    imgCode.id = res.id
    imgCode.code = res.code
    form.verify_id = res.id
  })
}
getImgCode()
// 获取短信验证码
const getSmsCode = () => {
  formRef.value.validateField(['email', 'verify_code'], (valid) => {
    if (valid) {
      sms_loading.value = true
      verifyCode.smsSend({
        account: email.value,
        verify_id: form.verify_id,
        verify_code: form.verify_code,
        type: 'email'
      }).then(res => {
        sms_loading.value = false
        ctx.$message({ message: '验证码发送成功', duration: 1000, type: 'success' })
        imgCode.msg = '验证码发送成功'
        let timer = setInterval(() => {
          if (imgCode.counter <= 0) {
            clearInterval(timer)
            imgCode.msg = '重新获取验证码'
            imgCode.counter = 60
          } else {
            imgCode.counter--
            imgCode.msg = `${imgCode.counter}秒后可重发`
          }
        }, 1000)
      }).catch((error) => {
        ctx.$message({ message: error.message, type: 'error', duration: 1000 })
        sms_loading.value = false
      })
    }
  })
}
</script>

<style lang="scss">
.el-form.login-form{
  padding: 10px 0 25px;
  .el-input__inner{
    background-color: rgba(255,255,255,0.8);
    // font-size: 17px !important;
  }
  .el-select {
    .el-input--suffix {
      .el-input__wrapper {
        padding: 1px 8px !important;
      }
      .el-input__inner {
        background-color: transparent !important;
      }
    }
  }
  .register-enter {
    margin: 10px 0;
    .el-form-item__content {
      justify-content: space-between;
      color: #40485b;
      a {
        color: #409eff;
        text-decoration: none;
        cursor: pointer;
      }
    }
  }
  .verify-code {
    .el-form-item__content {
      justify-content: space-between;
      flex-wrap: nowrap;
      img {
        height: 40px;
        margin-left: 10px;
        border-radius: 4px;
        flex-shrink: 0;
        cursor: pointer;
      }
    }
  }
  .email-code {
    .el-input__wrapper {
      padding-right: 4px !important;
    }
    .el-button {
      padding: 8px 13px;
    }
  }
}
</style>