<template>
  <div class="card" style="max-width:700px;margin:0 auto;">
    <h1 style="margin-bottom:8px;">{{ isEdit ? '编辑盲盒' : '发布盲盒' }}</h1>
    <p style="color:#666;margin-bottom:24px;">{{ isEdit ? '修改你的盲盒信息' : '填写以下信息，发布你的神秘盲盒' }}</p>

    <form @submit.prevent="handleSubmit">
      <div class="form-group">
        <label>物品类别 *</label>
        <select v-model="form.category" required>
          <option value="">请选择类别</option>
          <option value="book">书籍类</option>
          <option value="figure">手办类</option>
          <option value="toy">玩具类</option>
          <option value="game">游戏类</option>
          <option value="digital">数码类</option>
          <option value="other">其他</option>
        </select>
      </div>

      <div class="form-group">
        <label>神秘标签 *（用逗号分隔）</label>
        <input v-model="tagsInput" placeholder="例如：全新,限量,珍藏版" required />
        <small style="color:#999;">其他用户只能看到这些标签来猜测物品内容</small>
      </div>

      <div class="form-group">
        <label>真实物品名称 *（仅交换成功后对方可见）</label>
        <input v-model="form.realName" placeholder="例如：《三体》全集精装版" required />
      </div>

      <div class="form-group">
        <label>成色 *</label>
        <select v-model="form.condition" required>
          <option value="">请选择成色</option>
          <option value="全新">全新</option>
          <option value="九五成新">九五成新</option>
          <option value="九成新">九成新</option>
          <option value="八成新">八成新</option>
          <option value="七成新及以下">七成新及以下</option>
          <option value="有瑕疵">有瑕疵</option>
        </select>
        <small style="color:#999;">客观描述物品的新旧程度，便于其他用户判断</small>
      </div>

      <div class="form-group">
        <label>包装/配件完整度 *</label>
        <select v-model="form.completeness" required>
          <option value="">请选择完整度</option>
          <option value="包装配件齐全">包装配件齐全</option>
          <option value="有包装无配件">有包装无配件</option>
          <option value="有配件无包装">有配件无包装</option>
          <option value="无包装无配件">无包装无配件</option>
          <option value="仅本体">仅本体</option>
        </select>
        <small style="color:#999;">描述物品包装和配件的完整情况</small>
      </div>

      <div class="form-group">
        <label>物品详细描述（仅交换成功后对方可见）</label>
        <textarea v-model="form.description" rows="4" placeholder="描述物品的具体特点、使用情况等信息"></textarea>
      </div>

      <div class="form-group">
        <label>物品图片 {{ isEdit ? '' : '*' }}</label>
        <input type="file" accept="image/*" @change="handleFileChange" :required="!isEdit" />
        <div v-if="previewUrl" style="margin-top:12px;">
          <img :src="previewUrl" style="max-width:200px;border-radius:8px;"/>
        </div>
        <small style="color:#999;">图片会模糊处理展示给其他用户，交换成功后才会显示原图</small>
      </div>

      <div class="form-group">
        <label>联系方式 *（交换成功后对方可见）</label>
        <input v-model="form.contact" placeholder="例如：微信 xxx 或 QQ xxx" required />
      </div>

      <div style="display:flex;gap:12px;">
        <button type="submit" class="btn btn-primary" style="flex:1;" :disabled="submitting">
          {{ submitting ? (isEdit ? '保存中...' : '发布中...') : (isEdit ? '保存修改' : '发布盲盒') }}
        </button>
        <router-link :to="isEdit ? '/my-items' : '/'" style="flex:1;">
          <button type="button" class="btn btn-secondary" style="width:100%;">取消</button>
        </router-link>
      </div>
    </form>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRouter, useRoute } from 'vue-router'
import { createItem, updateItem, getItemDetail } from '../api/index.js'
import { userStore } from '../store/user.js'
import { appendAuth } from '../api/index.js'

const router = useRouter()
const route = useRoute()

const isEdit = computed(function() {
  return route.name === 'Edit'
})

const editItemId = computed(function() {
  return route.params.id || null
})

const form = ref({
  category: '',
  realName: '',
  description: '',
  contact: '',
  condition: '',
  completeness: ''
})

const tagsInput = ref('')
const imageFile = ref(null)
const previewUrl = ref('')
const submitting = ref(false)
const loading = ref(false)

const mysteryTags = computed(function() {
  return tagsInput.value
    .split(/[,，]/)
    .map(function(t) { return t.trim() })
    .filter(function(t) { return t.length > 0 })
})

function handleFileChange(e) {
  const file = e.target.files[0]
  if (file) {
    imageFile.value = file
    previewUrl.value = URL.createObjectURL(file)
  }
}

async function loadItemForEdit() {
  if (!isEdit.value || !editItemId.value) return

  loading.value = true
  try {
    const item = await getItemDetail(editItemId.value, userStore.user.id)
    form.value.category = item.category || ''
    form.value.realName = item.realName || ''
    form.value.description = item.description || ''
    form.value.contact = item.contact || ''
    form.value.condition = item.condition || ''
    form.value.completeness = item.completeness || ''
    tagsInput.value = (item.mysteryTags || []).join(',')
    if (item.image) {
      previewUrl.value = appendAuth(item.image)
    }
  } catch (e) {
    alert('加载物品信息失败')
    router.push('/my-items')
  } finally {
    loading.value = false
  }
}

async function handleSubmit() {
  if (mysteryTags.value.length === 0) {
    alert('请至少填写一个神秘标签')
    return
  }
  if (!form.value.condition) {
    alert('请选择物品成色')
    return
  }
  if (!form.value.completeness) {
    alert('请选择包装/配件完整度')
    return
  }
  if (!isEdit.value && !imageFile.value) {
    alert('请上传物品图片')
    return
  }

  submitting.value = true
  try {
    const formData = new FormData()
    formData.append('category', form.value.category)
    formData.append('mysteryTags', JSON.stringify(mysteryTags.value))
    formData.append('realName', form.value.realName)
    formData.append('description', form.value.description)
    formData.append('contact', form.value.contact)
    formData.append('condition', form.value.condition)
    formData.append('completeness', form.value.completeness)
    formData.append('ownerId', userStore.user.id)
    formData.append('ownerName', userStore.user.name)
    if (imageFile.value) {
      formData.append('image', imageFile.value)
    }

    if (isEdit.value) {
      await updateItem(editItemId.value, formData)
      alert('修改成功！')
      router.push('/my-items')
    } else {
      await createItem(formData)
      alert('发布成功！')
      router.push('/my-items')
    }
  } catch (e) {
    const msg = e.response && e.response.data ? e.response.data.error : e.message
    alert((isEdit.value ? '修改失败：' : '发布失败：') + msg)
  } finally {
    submitting.value = false
  }
}

onMounted(loadItemForEdit)
</script>
