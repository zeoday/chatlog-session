<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { useContactStore } from '@/stores/contact'
import { useRouter } from 'vue-router'
import Avatar from '@/components/common/Avatar.vue'
import SearchBar from '@/components/common/SearchBar.vue'
import Loading from '@/components/common/Loading.vue'
import Empty from '@/components/common/Empty.vue'
import Error from '@/components/common/Error.vue'
import type { Contact } from '@/types'
import { ContactType } from '@/types/contact'

const contactStore = useContactStore()
const router = useRouter()

// 状态
const loading = ref(false)
const error = ref<Error | null>(null)
const searchText = ref('')
const filterType = ref<'all' | 'friends' | 'groups' | 'starred'>('all')
const sortBy = ref<'name' | 'pinyin'>('pinyin')

// 计算属性
const filteredContacts = computed(() => {
  // 确保 contacts 是数组
  const allContacts = Array.isArray(contactStore.contacts)
    ? contactStore.contacts
    : []

  let contacts = allContacts

  // 按类型筛选
  switch (filterType.value) {
    case 'friends':
      contacts = allContacts.filter(c => c.type === ContactType.Friend)
      break
    case 'groups':
      contacts = allContacts.filter(c => c.type === ContactType.Chatroom)
      break
    case 'starred':
      contacts = allContacts.filter(c => c.isStarred === true)
      break
  }

  // 搜索过滤
  if (searchText.value) {
    const keyword = searchText.value.toLowerCase()
    contacts = contacts.filter(contact =>
      contact.nickname.toLowerCase().includes(keyword) ||
      contact.alias?.toLowerCase().includes(keyword) ||
      contact.remark?.toLowerCase().includes(keyword) ||
      contact.wxid?.toLowerCase().includes(keyword)
    )
  }

  return contacts
})

// 按首字母分组
const groupedContacts = computed(() => {
  if (sortBy.value === 'name') {
    return { '全部': filteredContacts.value }
  }
  // 简单按首字母分组
  const grouped: Record<string, Contact[]> = {}
  filteredContacts.value.forEach(contact => {
    const initial = contact.nickname.charAt(0).toUpperCase()
    if (!grouped[initial]) {
      grouped[initial] = []
    }
    grouped[initial].push(contact)
  })
  return grouped
})

// 统计信息
const stats = computed(() => {
  const allContacts = Array.isArray(contactStore.contacts)
    ? contactStore.contacts
    : []

  return {
    total: allContacts.length,
    friends: allContacts.filter(c => c.type === ContactType.Friend).length,
    groups: allContacts.filter(c => c.type === ContactType.Chatroom).length,
    starred: allContacts.filter(c => c.isStarred === true).length
  }
})

// 加载联系人
const loadContacts = async () => {
  loading.value = true
  error.value = null

  try {
    await contactStore.loadContacts()
    //await contactStore.loadChatrooms()
  } catch (e: any) {
    error.value = e
    console.error('加载联系人失败:', e)
  } finally {
    loading.value = false
  }
}

// 搜索处理
const handleSearch = (value: string) => {
  searchText.value = value
}

// 查看联系人详情
const viewContact = (contact: Contact) => {
  console.log('查看联系人:', contact)
  // TODO: 打开联系人详情弹窗或跳转详情页
}

// 发起聊天
const startChat = (contact: Contact) => {
  // 跳转到聊天页面并选择该联系人的会话
  router.push({
    path: '/chat',
    query: { talker: contact.wxid }
  })
}

// 刷新
const handleRefresh = () => {
  loadContacts()
}

// 初始化
onMounted(() => {
  if (contactStore.contacts.length === 0) {
    loadContacts()
  }
})
</script>

<template>
  <div class="contact-page">
    <div class="contact-container">
      <!-- 左侧：联系人列表 -->
      <div class="contact-list-panel">
        <!-- 头部 -->
        <div class="contact-header">
          <div class="header-title">
            <h2>联系人</h2>
            <el-tag v-if="stats.total > 0" size="small" type="info">
              {{ stats.total }}
            </el-tag>
          </div>

          <!-- 搜索框 -->
          <SearchBar
            v-model="searchText"
            placeholder="搜索联系人"
            size="default"
            class="contact-search"
            @search="handleSearch"
          />

          <!-- 筛选和排序 -->
          <div class="contact-filters">
            <el-radio-group v-model="filterType" size="small">
              <el-radio-button label="all">
                全部 ({{ stats.total }})
              </el-radio-button>
              <el-radio-button label="friends">
                好友 ({{ stats.friends }})
              </el-radio-button>
              <el-radio-button label="groups">
                群聊 ({{ stats.groups }})
              </el-radio-button>
              <el-radio-button label="starred">
                星标 ({{ stats.starred }})
              </el-radio-button>
            </el-radio-group>

            <el-button-group size="small" class="sort-buttons">
              <el-button
                :type="sortBy === 'pinyin' ? 'primary' : 'default'"
                @click="sortBy = 'pinyin'"
              >
                <el-icon><Sort /></el-icon>
                字母
              </el-button>
              <el-button
                :type="sortBy === 'name' ? 'primary' : 'default'"
                @click="sortBy = 'name'"
              >
                <el-icon><List /></el-icon>
                列表
              </el-button>
            </el-button-group>
          </div>
        </div>

        <!-- 加载状态 -->
        <Loading v-if="loading" text="加载联系人中..." />

        <!-- 错误状态 -->
        <Error
          v-else-if="error"
          title="加载失败"
          :error="error"
          @retry="handleRefresh"
        />

        <!-- 空状态 -->
        <Empty
          v-else-if="filteredContacts.length === 0"
          icon="User"
          :description="searchText ? '未找到匹配的联系人' : '暂无联系人'"
        >
          <el-button v-if="!searchText" type="primary" @click="handleRefresh">
            <el-icon><Refresh /></el-icon>
            刷新
          </el-button>
        </Empty>

        <!-- 联系人列表 -->
        <div v-else class="contact-list-container">
          <el-scrollbar>
            <div
              v-for="(contacts, initial) in groupedContacts"
              :key="initial"
              class="contact-group"
            >
              <!-- 分组标题 -->
              <div class="group-header">{{ initial }}</div>

              <!-- 联系人项 -->
              <div
                v-for="contact in contacts"
                :key="contact.wxid"
                class="contact-item"
                @click="viewContact(contact)"
              >
                <Avatar
                  :src="contact.avatar"
                  :name="contact.nickname"
                  :size="48"
                  class="contact-avatar"
                />

                <div class="contact-info">
                  <div class="contact-name">
                    <span class="name-text">{{ contact.remark || contact.nickname }}</span>
                    <el-icon v-if="contact.isStarred" color="#f59e0b" size="16">
                      <StarFilled />
                    </el-icon>
                  </div>
                  <div class="contact-desc">
                    <el-tag
                      v-if="contact.type"
                      size="small"
                      :type="contact.type === ContactType.Chatroom ? 'warning' : 'info'"
                      effect="plain"
                    >
                      {{ contact.type === ContactType.Chatroom ? '群聊' : '好友' }}
                    </el-tag>
                    <span v-if="contact.alias" class="alias">{{ contact.alias }}</span>
                  </div>
                </div>

                <div class="contact-actions">
                  <el-button
                    text
                    type="primary"
                    size="small"
                    @click.stop="startChat(contact)"
                  >
                    <el-icon><ChatDotRound /></el-icon>
                    发消息
                  </el-button>
                </div>
              </div>
            </div>
          </el-scrollbar>
        </div>
      </div>

      <!-- 右侧：联系人详情 -->
      <div class="contact-detail-panel">
        <el-empty
          description="选择一个联系人查看详情"
          :image-size="160"
        >
          <template #image>
            <el-icon size="160" color="#909399">
              <UserFilled />
            </el-icon>
          </template>
        </el-empty>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.contact-page {
  width: 100%;
  height: 100%;
  background-color: var(--el-bg-color);
  overflow: hidden;
}

.contact-container {
  display: flex;
  width: 100%;
  height: 100%;
}

// 联系人列表面板
.contact-list-panel {
  width: 380px;
  height: 100%;
  background-color: var(--el-bg-color);
  border-right: 1px solid var(--el-border-color-light);
  display: flex;
  flex-direction: column;
  flex-shrink: 0;

  .contact-header {
    padding: 16px;
    border-bottom: 1px solid var(--el-border-color-lighter);
    flex-shrink: 0;

    .header-title {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-bottom: 12px;

      h2 {
        margin: 0;
        font-size: 20px;
        font-weight: 600;
      }
    }

    .contact-search {
      margin-bottom: 12px;
    }

    .contact-filters {
      display: flex;
      flex-direction: column;
      gap: 8px;

      .el-radio-group {
        width: 100%;

        :deep(.el-radio-button) {
          flex: 1;

          .el-radio-button__inner {
            width: 100%;
            font-size: 12px;
            padding: 8px 4px;
          }
        }
      }

      .sort-buttons {
        width: 100%;

        .el-button {
          flex: 1;
        }
      }
    }
  }

  .contact-list-container {
    flex: 1;
    overflow: hidden;

    .contact-group {
      .group-header {
        padding: 8px 16px;
        font-size: 12px;
        font-weight: 600;
        color: var(--el-text-color-secondary);
        background-color: var(--el-fill-color-light);
        position: sticky;
        top: 0;
        z-index: 10;
      }

      .contact-item {
        display: flex;
        align-items: center;
        gap: 12px;
        padding: 12px 16px;
        cursor: pointer;
        transition: all 0.2s;
        border-bottom: 1px solid var(--el-border-color-lighter);

        &:hover {
          background-color: var(--el-fill-color-light);

          .contact-actions {
            opacity: 1;
          }
        }

        .contact-avatar {
          flex-shrink: 0;
        }

        .contact-info {
          flex: 1;
          min-width: 0;

          .contact-name {
            display: flex;
            align-items: center;
            gap: 4px;
            margin-bottom: 4px;

            .name-text {
              font-size: 14px;
              font-weight: 500;
              color: var(--el-text-color-primary);
              overflow: hidden;
              text-overflow: ellipsis;
              white-space: nowrap;
            }
          }

          .contact-desc {
            display: flex;
            align-items: center;
            gap: 8px;
            font-size: 12px;

            .alias {
              color: var(--el-text-color-secondary);
              overflow: hidden;
              text-overflow: ellipsis;
              white-space: nowrap;
            }
          }
        }

        .contact-actions {
          flex-shrink: 0;
          opacity: 0;
          transition: opacity 0.2s;
        }
      }
    }
  }
}

// 联系人详情面板
.contact-detail-panel {
  flex: 1;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: var(--el-bg-color);
  min-width: 0;
}

// 响应式
@media (max-width: 768px) {
  .contact-container {
    flex-direction: column;
  }

  .contact-list-panel {
    width: 100%;
    border-right: none;
    border-bottom: 1px solid var(--el-border-color-light);
  }

  .contact-detail-panel {
    display: none;
  }
}
</style>
