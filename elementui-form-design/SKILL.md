---
name: elementui-form-design
description: |
  Generates Vue form pages using Element UI components. Use this skill whenever the user wants to create form pages, edit forms, or generate form code that uses Element UI. Trigger when they say things like "generate a form page", "I need a form page", "create a user form", "make an edit form", or any request for form page generation.
compatibility:
  required_tools: []
  dependencies: []
---

# Element UI Form Page Generator

Generates complete, production-ready Vue form pages using Element UI components. This skill creates form structures with data binding, validation, submit/reset handlers, and follows Vue 3 Composition API patterns.

## What This Skill Does

When a user asks for a form page, this skill:
1. Gathers requirements about the form (field types, validation rules, purpose)
2. Generates a complete `.vue` file with Element UI components
3. Includes data models, form validation, and event handlers
4. Provides clean, maintainable code following Vue 3 best practices

## How to Use This Skill

### Step 1: Understand the Form Requirements
When the user asks for a form, ask clarifying questions if needed:
- **Form purpose**: What is this form for? (login, registration, user profile edit, data entry, etc.)
- **Fields needed**: What fields should the form contain? (name, email, phone, date, dropdown, etc.)
- **Validation**: Are there specific validation rules? (required, email format, length, custom rules)
- **Actions**: Should it have submit/reset buttons? Where should it submit?
- **Layout**: Any specific layout preferences? (single column, two columns, etc.)

### Step 2: Generate the .vue File

Create a Vue component with:
- **Template section**: Form UI using Element UI `<el-form>` and form controls
- **Script section**: Data model, validation rules, and event handlers using Composition API
- **Style section**: Basic styling if needed

### Step 3: Use Element UI Components

Common components to include:
- `<el-form>` - Form wrapper
- `<el-form-item>` - Field wrapper with label
- `<el-input>` - Text input
- `<el-input-number>` - Number input
- `<el-select>` - Dropdown
- `<el-date-picker>` - Date selection
- `<el-checkbox>` - Checkbox
- `<el-radio>` - Radio button
- `<el-switch>` - Toggle switch
- `<el-button>` - Button

### Step 4: Include Form Logic

Every generated form should include:
- **Data binding** with reactive() or ref()
- **Form rules** for validation
- **resetForm()** method to clear the form
- **submitForm()** method to handle submission
- **Form ref** for programmatic access

## Code Pattern

vue3写法
```vue
<template>
  <div class="form-container">
    <el-form ref="formRef" :model="formData" :rules="rules" label-width="120px">
      <!-- form items here -->
      <el-form-item>
        <el-button type="primary" @click="submitForm">Submit</el-button>
        <el-button @click="resetForm">Reset</el-button>
      </el-form-item>
    </el-form>
  </div>
</template>

<script setup lang="ts">
import { reactive, ref } from 'vue'
import { ElMessage } from 'element-plus'

const formRef = ref()
const formData = reactive({
  // fields here
})

const rules = {
  // validation rules here
}

const submitForm = () => {
  formRef.value?.validate((valid: boolean) => {
    if (valid) {
      ElMessage.success('Form submitted successfully')
      // handle submission
    }
  })
}

const resetForm = () => {
  formRef.value?.resetFields()
}
</script>

<style scoped>
.form-container {
  padding: 20px;
}
</style>
```

vue2写法
```vue
<el-form :model="ruleForm" status-icon :rules="rules" ref="ruleForm" label-width="100px" class="demo-ruleForm">
  <el-form-item label="密码" prop="pass">
    <el-input type="password" v-model="ruleForm.pass" autocomplete="off"></el-input>
  </el-form-item>
  <el-form-item label="确认密码" prop="checkPass">
    <el-input type="password" v-model="ruleForm.checkPass" autocomplete="off"></el-input>
  </el-form-item>
  <el-form-item label="年龄" prop="age">
    <el-input v-model.number="ruleForm.age"></el-input>
  </el-form-item>
  <el-form-item>
    <el-button type="primary" @click="submitForm('ruleForm')">提交</el-button>
    <el-button @click="resetForm('ruleForm')">重置</el-button>
  </el-form-item>
</el-form>
<script>
  export default {
    data() {
      var validatePass = (rule, value, callback) => {
        if (value === '') {
          callback(new Error('请输入密码'));
        } else {
          if (this.ruleForm.checkPass !== '') {
            this.$refs.ruleForm.validateField('checkPass');
          }
          callback();
        }
      };
      var validatePass2 = (rule, value, callback) => {
        if (value === '') {
          callback(new Error('请再次输入密码'));
        } else if (value !== this.ruleForm.pass) {
          callback(new Error('两次输入密码不一致!'));
        } else {
          callback();
        }
      };
      return {
        ruleForm: {
          pass: '',
          checkPass: '',
          age: ''
        },
        rules: {
          pass: [
            { validator: validatePass, trigger: 'blur' }
          ],
          checkPass: [
            { validator: validatePass2, trigger: 'blur' }
          ],
          age: [
            { required: true, message: '请填写年龄', trigger: 'blur' }
          ]
        }
      };
    },
    methods: {
      submitForm(formName) {
        this.$refs[formName].validate((valid) => {
          if (valid) {
            alert('submit!');
          } else {
            console.log('error submit!!');
            return false;
          }
        });
      },
      resetForm(formName) {
        this.$refs[formName].resetFields();
      }
    }
  }
</script>

```

## Tips for Better Forms

1. **Validation**: Use Element UI's built-in validation rules (required, email, url, etc.)
2. **Placeholders**: Add helpful placeholder text to inputs
3. **Required fields**: Mark required fields clearly in labels
4. **Error messages**: Use descriptive validation error messages
5. **Responsive**: Consider form layout for different screen sizes
6. **User feedback**: Use ElMessage for success/error feedback

## Example Trigger Phrases

- "Generate a form page for user registration"
- "I need an edit form for products"
- "Create a contact form page"
- "Make a form with name, email, and phone fields"
- "Help me build a form page using Element UI"
