<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { useChatStore } from '@/stores/chat'
import { useSessionStore } from '@/stores/session'
import { useRouter } from 'vue-router'
import { ElMessage } from 'element-plus'
import Avatar from '@/components/common/Avatar.vue'
import SearchBar from '@/components/common/SearchBar.vue'
import Loading from '@/components/common/Loading.vue'
import Empty from '@/components/common/Empty.vue'
import MessageBubble from '@/components/chat/MessageBubble.vue'
import type { Message } from '@/types'

const chatStore = useChatStore()
const sessionStore = useSessionStore()
const router = useRouter()

// 状态
const loading = ref(false)
const searchText = ref('')
const searchType = ref<'all' | 'message' | 'session'>('all')
const searchScope = ref<'global' | 'session'>('global')
const selectedSessionId = ref<string>('')
const dateRange = ref<[Date, Date] | null>(null)

// 搜索结果
const messageResults = ref<Message[]>([])
const sessionResults = ref<any[]>([])

// 计算属性
const hasResults = computed(() => {
  return messageResults.value.length > 0 || sessionResults.value.length > 0
})

const filteredMessageResults = computed(() => {
  if (searchType.value === 'session') return []
  return messageResults.value
})

const filteredSessionResults = computed(() => {
  if (searchType.value === 'message') return []
  return sessionResults.value
})

const stats = computed(() => ({
  messages: messageResults.value.length,
  sessions: sessionResults.value.length,
  total: messageResults.value.length + sessionResults.value.length
}))

// 执行搜索
const performSearch = async () => {
  if (!searchText.value.trim()) {
    messageResults.value = []
    sessionResults.value = []
    return
  }

  loading.value = true

  try {
    // 搜索消息
    if (searchType.value === 'all' || searchType.value === 'message') {
      if (searchScope.value === 'global') {
        // 全局搜索消息
        await chatStore.searchMessages(searchText.value)
        messageResults.value = chatStore.searchResults
      } else if (selectedSessionId.value) {
        // 会话内搜索
        // 会话内搜索暂时使用全局搜索
        await chatStore.searchMessages(searchText.value)
        messageResults.value = chatStore.searchResults
      }
    }

    // 搜索会话
    if (searchType.value === 'all' || searchType.value === 'session') {
      // 会话搜索暂时在前端过滤
      const keyword = searchText.value.toLowerCase()
      sessionResults.value = sessionStore.sessions.filter(s => 
        s.name?.toLowerCase().includes(keyword) ||
        s.talkerName?.toLowerCase().includes(keyword)
      )
    }
  } catch (error) {
    console.error('搜索失败:', error)
    ElMessage.error('搜索失败，请重试')
  } finally {
    loading.value = false
  }
}

// 处理搜索
const handleSearch = (value: string) => {
  searchText.value = value
  performSearch()
}

// 清空搜索
const clearSearch = () => {
  searchText.value = ''
  messageResults.value = []
  sessionResults.value = []
}

// 查看消息
const viewMessage = (message: Message) => {
  // 跳转到聊天页面并定位到该消息
  router.push({
    path: '/chat',
    query: {
      talker: message.talker,
      messageId: message.id
    }
  })
}

// 查看会话
const viewSession = (session: any) => {
  router.push({
    path: '/chat',
    query: { talker: session.talker }
  })
}

// 监听搜索类型变化
watch(searchType, () => {
  if (searchText.value) {
    performSearch()
  }
})

// 监听搜索范围变化
watch(searchScope, () => {
  if (searchText.value) {
    performSearch()
  }
})

// 监听选中会话变化
watch(selectedSessionId, () => {
  if (searchText.value && searchScope.value === 'session') {
    performSearch()
  }
})
</script>

<template>
  <div class="search-page">
    <div class="search-container">
      <!-- 搜索头部 -->
      <div class="search-header">
        <div class="header-top">
          <el-button
            text
            size="large"
            @click="$router.back()"
          >
            <el-icon><ArrowLeft /></el-icon>
          </el-button>
          <h2>搜索</h2>
        </div>

        <!-- 搜索框 -->
        <SearchBar
          v-model="searchText"
          placeholder="搜索消息或会话"
          size="large"
          autofocus
          class="main-search"
          @search="handleSearch"
        />

        <!-- 搜索选项 -->
        <div class="search-options">
          <div class="option-row">
            <label>搜索类型:</label>
            <el-radio-group v-model="searchType" size="small">
              <el-radio-button label="all">全部</el-radio-button>
              <el-radio-button label="message">消息</el-radio-button>
              <el-radio-button label="session">会话</el-radio-button>
            </el-radio-group>
          </div>

          <div v-if="searchType !== 'session'" class="option-row">
            <label>搜索范围:</label>
            <el-radio-group v-model="searchScope" size="small">
              <el-radio-button label="global">全局</el-radio-button>
              <el-radio-button label="session">当前会话</el-radio-button>
            </el-radio-group>
          </div>

          <div v-if="searchScope === 'session' && searchType !== 'session'" class="option-row">
            <label>选择会话:</label>
            <el-select
              v-model="selectedSessionId"
              placeholder="请选择会话"
              size="small"
              filterable
              style="flex: 1"
            >
              <el-option
                v-for="session in sessionStore.sessions"
                :key="session.id"
                :label="session.name"
                :value="session.id"
              >
                <div style="display: flex; align-items: center; gap: 8px;">
                  <Avatar :src="session.avatar" :name="session.name" :size="24" />
                  <span>{{ session.name }}</span>
                </div>
              </el-option>
            </el-select>
          </div>

          <!-- 日期范围 -->
          <div v-if="searchType !== 'session'" class="option-row">
            <label>日期范围:</label>
            <el-date-picker
              v-model="dateRange"
              type="daterange"
              range-separator="至"
              start-placeholder="开始日期"
              end-placeholder="结束日期"
              size="small"
              style="flex: 1"
            />
          </div>
        </div>
      </div>

      <!-- 搜索结果 -->
      <div class="search-content">
        <!-- 加载状态 -->
        <Loading v-if="loading" text="搜索中..." />

        <!-- 空状态 -->
        <Empty
          v-else-if="!searchText"
          icon="Search"
          description="输入关键词开始搜索"
        />

        <!-- 无结果 -->
        <Empty
          v-else-if="!loading && !hasResults"
          icon="DocumentDelete"
          description="未找到相关结果"
        >
          <el-button type="primary" @click="clearSearch">
            清空搜索
          </el-button>
        </Empty>

        <!-- 搜索结果 -->
        <div v-else class="search-results">
          <!-- 统计信息 -->
          <div class="result-stats">
            <el-tag type="info" size="large">
              共找到 {{ stats.total }} 条结果
            </el-tag>
            <el-tag v-if="stats.messages > 0" type="success" size="small">
              {{ stats.messages }} 条消息
            </el-tag>
            <el-tag v-if="stats.sessions > 0" type="warning" size="small">
              {{ stats.sessions }} 个会话
            </el-tag>
          </div>

          <el-scrollbar>
            <!-- 会话结果 -->
            <div v-if="filteredSessionResults.length > 0" class="result-section">
              <div class="section-header">
                <h3>会话</h3>
                <el-tag size="small">{{ filteredSessionResults.length }}</el-tag>
              </div>

              <div class="session-results">
                <div
                  v-for="session in filteredSessionResults"
                  :key="session.id"
                  class="session-item"
                  @click="viewSession(session)"
                >
                  <Avatar
                    :src="session.avatar"
                    :name="session.name"
                    :size="48"
                  />
                  <div class="session-info">
                    <div class="session-name">{{ session.name }}</div>
                    <div class="session-desc">
                      <el-tag
                        size="small"
                        :type="session.type === 'group' ? 'warning' : 'info'"
                        effect="plain"
                      >
                        {{ session.type === 'group' ? '群聊' : '私聊' }}
                      </el-tag>
                      <span v-if="session.lastMessage" class="last-message">
                        {{ session.lastMessage.content }}
                      </span>
                    </div>
                  </div>
                  <div class="session-actions">
                    <el-icon><ArrowRight /></el-icon>
                  </div>
                </div>
              </div>
            </div>

            <!-- 消息结果 -->
            <div v-if="filteredMessageResults.length > 0" class="result-section">
              <div class="section-header">
                <h3>消息</h3>
                <el-tag size="small">{{ filteredMessageResults.length }}</el-tag>
              </div>

              <div class="message-results">
                <div
                  v-for="message in filteredMessageResults"
                  :key="message.id"
                  class="message-item"
                  @click="viewMessage(message)"
                >
                  <div class="message-header">
                    <Avatar
                      :src="message.talkerAvatar"
                      :name="message.talkerName"
                      :size="32"
                    />
                    <div class="message-meta">
                      <span class="sender-name">{{ message.talkerName }}</span>
                      <span class="message-time">
                        {{ new Date(message.createTime).toLocaleString() }}
                      </span>
                    </div>
                  </div>

                  <div class="message-content">
                    <MessageBubble
                      :message="message"
                      :is-send="message.isSend"
                      :show-avatar="false"
                      class="search-bubble"
                    />
                  </div>
                </div>
              </div>
            </div>
          </el-scrollbar>
        </div>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
.search-page {
  width: 100%;
  height: 100%;
  background-color: var(--el-bg-color);
  overflow: hidden;
}

.search-container {
  max-width: 900px;
  margin: 0 auto;
  height: 100%;
  display: flex;
  flex-direction: column;
}

.search-header {
  padding: 16px 24px;
  border-bottom: 1px solid var(--el-border-color-light);
  flex-shrink: 0;
  background-color: var(--el-bg-color);

  .header-top {
    display: flex;
    align-items: center;
    gap: 12px;
    margin-bottom: 16px;

    h2 {
      margin: 0;
      font-size: 20px;
      font-weight: 600;
    }
  }

  .main-search {
    margin-bottom: 16px;
  }

  .search-options {
    display: flex;
    flex-direction: column;
    gap: 12px;

    .option-row {
      display: flex;
      align-items: center;
      gap: 12px;

      label {
        font-size: 14px;
        font-weight: 500;
        color: var(--el-text-color-regular);
        white-space: nowrap;
        min-width: 80px;
      }

      .el-radio-group {
        flex: 1;
      }
    }
  }
}

.search-content {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.search-results {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;

  .result-stats {
    padding: 16px 24px;
    border-bottom: 1px solid var(--el-border-color-lighter);
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
  }

  .result-section {
    padding: 16px 24px;

    .section-header {
      display: flex;
      align-items: center;
      gap: 8px;
      margin-bottom: 12px;

      h3 {
        margin: 0;
        font-size: 16px;
        font-weight: 600;
      }
    }

    .session-results {
      display: flex;
      flex-direction: column;
      gap: 8px;

      .session-item {
        display: flex;
        align-items: center;
        gap: 12px;
        padding: 12px;
        border-radius: 8px;
        cursor: pointer;
        transition: all 0.2s;
        border: 1px solid var(--el-border-color-lighter);

        &:hover {
          background-color: var(--el-fill-color-light);
          border-color: var(--el-color-primary);
        }

        .session-info {
          flex: 1;
          min-width: 0;

          .session-name {
            font-size: 14px;
            font-weight: 500;
            color: var(--el-text-color-primary);
            margin-bottom: 4px;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
          }

          .session-desc {
            display: flex;
            align-items: center;
            gap: 8px;
            font-size: 12px;

            .last-message {
              color: var(--el-text-color-secondary);
              overflow: hidden;
              text-overflow: ellipsis;
              white-space: nowrap;
            }
          }
        }

        .session-actions {
          color: var(--el-text-color-placeholder);
        }
      }
    }

    .message-results {
      display: flex;
      flex-direction: column;
      gap: 12px;

      .message-item {
        padding: 12px;
        border-radius: 8px;
        cursor: pointer;
        transition: all 0.2s;
        border: 1px solid var(--el-border-color-lighter);

        &:hover {
          background-color: var(--el-fill-color-light);
          border-color: var(--el-color-primary);
        }

        .message-header {
          display: flex;
          align-items: center;
          gap: 8px;
          margin-bottom: 8px;

          .message-meta {
            display: flex;
            align-items: center;
            gap: 8px;

            .sender-name {
              font-size: 14px;
              font-weight: 500;
              color: var(--el-text-color-primary);
            }

            .message-time {
              font-size: 12px;
              color: var(--el-text-color-secondary);
            }
          }
        }

        .message-content {
          .search-bubble {
            :deep(.message-bubble) {
              max-width: 100%;
            }
          }
        }
      }
    }
  }
}

// 响应式
@media (max-width: 768px) {
  .search-header {
    padding: 12px 16px;

    .search-options {
      .option-row {
        flex-direction: column;
        align-items: stretch;

        label {
          min-width: unset;
        }
      }
    }
  }

  .search-results {
    .result-section {
      padding: 12px 16px;
    }
  }
}
</style>