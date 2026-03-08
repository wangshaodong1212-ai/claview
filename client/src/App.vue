<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';
import { ElMessage, ElDialog, ElInput } from 'element-plus';
import MarkdownRenderer from './components/MarkdownRenderer.vue';
import ContributionGraph from './components/ContributionGraph.vue';
import ProjectPieChart from './components/ProjectPieChart.vue';
import RecentSessions from './components/RecentSessions.vue';
import TrendChart from './components/TrendChart.vue';
import ActivityStats from './components/ActivityStats.vue';

// API 基础 URL
const API_BASE = 'http://localhost:3333/api';

// 复制到剪贴板
const copyToClipboard = async (text: string, label?: string) => {
  try {
    await navigator.clipboard.writeText(text);
    ElMessage.success({
      message: label ? `${label} 已复制` : '已复制',
      duration: 1500,
      plain: true
    });
  } catch (err) {
    ElMessage.error('复制失败');
  }
};

// 全局搜索相关
const searchDialogVisible = ref(false);
const searchQuery = ref('');
const searchResults = ref<any[]>([]);
const searching = ref(false);
const selectedSearchResult = ref<any>(null);
const previewLoading = ref(false);
const previewMessages = ref<Message[]>([]);
const shortcutsHelpVisible = ref(false);

// 快捷键列表
const shortcuts = [
  { key: '⌘ K', description: '全局搜索', action: '唤起搜索框' },
  { key: 'Esc', description: '关闭弹窗', action: '关闭当前打开的对话框' },
  { key: '1/2/3', description: '切换视图', action: '1=仪表盘 2=项目浏览 3=全局搜索' },
  { key: '↑/↓', description: '导航', action: '在列表中上下移动' },
  { key: 'Enter', description: '确认', action: '选中当前项' },
];

// 执行搜索
const performSearch = async () => {
  if (!searchQuery.value.trim()) {
    searchResults.value = [];
    return;
  }

  searching.value = true;
  try {
    const response = await fetch(`${API_BASE}/search?q=${encodeURIComponent(searchQuery.value)}`);
    const data = await response.json();
    if (data.success) {
      searchResults.value = data.results;
    }
  } catch (error) {
    console.error('搜索失败:', error);
  } finally {
    searching.value = false;
  }
};

// 选择搜索结果
const selectSearchResult = (result: any, jumpToView = false) => {
  selectedSearchResult.value = result;

  if (jumpToView) {
    // 跳转到项目和会话详情视图
    const project = projects.value.find(p => p.path === result.projectPath);
    if (project) {
      currentView.value = 'projects';
      handleSelectProject(project);
      // 延迟获取会话列表后再选中会话
      setTimeout(() => {
        const session = sessions.value.find(s => s.id === result.sessionId);
        if (session) {
          handleSelectSession(session);
        }
      }, 100);
    }
    searchDialogVisible.value = false;
  } else {
    // 在预览窗格中显示
    loadPreviewMessages(result);
  }
};

// 加载预览消息
const loadPreviewMessages = async (result: any) => {
  previewLoading.value = true;
  previewMessages.value = [];
  try {
    const response = await fetch(`${API_BASE}/session-detail?filePath=${encodeURIComponent(result.sessionPath)}`);
    const data = await response.json();
    if (data.success) {
      previewMessages.value = data.messages;
    }
  } catch (error) {
    console.error('加载预览失败:', error);
  } finally {
    previewLoading.value = false;
  }
};

// 快捷键监听
const handleKeydown = (e: KeyboardEvent) => {
  // Cmd/Ctrl + K 唤起搜索
  if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
    e.preventDefault();
    searchDialogVisible.value = true;
    searchQuery.value = '';
    searchResults.value = [];
    setTimeout(() => {
      const input = document.querySelector('.search-input input') as HTMLInputElement;
      input?.focus();
    }, 100);
  }
  // ? 唤起快捷键帮助
  if (e.key === '?' && !searchDialogVisible.value) {
    e.preventDefault();
    shortcutsHelpVisible.value = true;
  }
  // 数字键切换视图
  if (['1', '2', '3'].includes(e.key) && !e.metaKey && !e.ctrlKey) {
    const viewMap: Record<string, ViewType> = { '1': 'dashboard', '2': 'projects', '3': 'search' };
    const newView = viewMap[e.key];
    if (newView) {
      currentView.value = newView;
    }
  }
  // ESC 关闭弹窗
  if (e.key === 'Escape') {
    if (searchDialogVisible.value) {
      searchDialogVisible.value = false;
    }
    if (shortcutsHelpVisible.value) {
      shortcutsHelpVisible.value = false;
    }
  }
};

// 数据类型定义
interface Project {
  path: string;
  name: string;
  lastSessionId: string | null;
}

interface Session {
  id: string;
  filename: string;
  path: string;
  modifiedAt: Date;
}

interface ToolCall {
  id: string;
  type: string;
  function: {
    name: string;
    arguments: string;
  };
}

interface Message {
  role: 'user' | 'assistant';
  content: string;
  toolCalls: ToolCall[];
}

// 响应式状态
const projects = ref<Project[]>([]);
const sessions = ref<Session[]>([]);
const messages = ref<Message[]>([]);
const selectedProject = ref<Project | null>(null);
const selectedSession = ref<Session | null>(null);
const loadingProjects = ref(false);
const loadingSessions = ref(false);
const loadingMessages = ref(false);
const sessionTitles = ref<Record<string, string>>({});

// 视图类型
type ViewType = 'dashboard' | 'projects' | 'search';
const currentView = ref<ViewType>('projects');

// 导航菜单项
const navItems = [
  { key: 'dashboard' as ViewType, label: '仪表盘', icon: '📊' },
  { key: 'projects' as ViewType, label: '项目浏览', icon: '📁' },
  { key: 'search' as ViewType, label: '全局搜索', icon: '🔍' }
  // { key: 'security' as ViewType, label: '安全清理', icon: '🔒' }
];

// 统计数据
const stats = ref<any>(null);
const loadingStats = ref(false);

const fetchStats = async () => {
  loadingStats.value = true;
  try {
    const response = await fetch(`${API_BASE}/stats`);
    const data = await response.json();
    if (data.success) {
      stats.value = data.stats;
    }
  } catch (error) {
    console.error('获取统计失败:', error);
  } finally {
    loadingStats.value = false;
  }
};

// 获取项目列表
const fetchProjects = async () => {
  loadingProjects.value = true;
  try {
    const response = await fetch(`${API_BASE}/projects`);
    const data = await response.json();
    if (data.success) {
      projects.value = data.projects;
    }
  } catch (error) {
    console.error('获取项目列表失败:', error);
  } finally {
    loadingProjects.value = false;
  }
};

// 获取会话列表
const fetchSessions = async (projectPath: string) => {
  loadingSessions.value = true;
  try {
    const response = await fetch(`${API_BASE}/sessions?path=${encodeURIComponent(projectPath)}`);
    const data = await response.json();
    if (data.success) {
      sessions.value = data.sessions;
    }
  } catch (error) {
    console.error('获取会话列表失败:', error);
  } finally {
    loadingSessions.value = false;
  }
};

// 获取会话详情
const fetchSessionDetail = async (filePath: string) => {
  loadingMessages.value = true;
  try {
    const response = await fetch(`${API_BASE}/session-detail?filePath=${encodeURIComponent(filePath)}`);
    const data = await response.json();
    if (data.success) {
      messages.value = data.messages;
      // 自动生成会话标题
      if (selectedSession.value && !sessionTitles.value[selectedSession.value.id]) {
        sessionTitles.value[selectedSession.value.id] = generateSessionTitle(data.messages);
      }
    }
  } catch (error) {
    console.error('获取会话详情失败:', error);
  } finally {
    loadingMessages.value = false;
  }
};

// 智能生成会话标题
const generateSessionTitle = (messages: Message[]): string => {
  if (!messages || messages.length === 0) {
    return '新对话';
  }

  // 获取前几条用户消息
  const userMessages = messages.filter(m => m.role === 'user').slice(0, 2);
  const firstUserMessage = userMessages[0]?.content || '';

  // 提取关键信息生成标题
  let title = '';

  // 尝试识别常见任务类型
  const taskPatterns = [
    { pattern: /创建|新建|添加|generate|create/i, prefix: '创建' },
    { pattern: /修复|解决|fix|solve|debug/i, prefix: '修复' },
    { pattern: /优化|改进|improve|optimize/i, prefix: '优化' },
    { pattern: /实现|完成|implement|complete/i, prefix: '实现' },
    { pattern: /删除|移除|delete|remove/i, prefix: '删除' },
    { pattern: /更新|修改|修改|update|modify|change/i, prefix: '更新' },
    { pattern: /搜索|查找|search|find/i, prefix: '搜索' },
    { pattern: /分析|了解|analyze|understand/i, prefix: '分析' },
  ];

  for (const { pattern, prefix } of taskPatterns) {
    if (pattern.test(firstUserMessage)) {
      title = prefix;
      break;
    }
  }

  // 提取关键词
  const cleanText = firstUserMessage
    .replace(/[^\u4e00-\u9fa5a-zA-Z0-9\s]/g, ' ')
    .trim()
    .substring(0, 50);

  const keywords = cleanText.split(/\s+/).filter(w => w.length > 1);

  // 组合标题
  if (!title) {
    title = keywords[0] || '对话';
  } else if (keywords.length > 0) {
    const keyword = keywords.find(k => k.length <= 8) || keywords[0];
    title += ' ' + keyword;
  }

  // 限制长度
  if (title.length > 20) {
    title = title.substring(0, 20) + '...';
  }

  return title || '新对话';
};

// 获取会话标题
const getSessionTitle = (sessionId: string): string => {
  return sessionTitles.value[sessionId] || sessionId.slice(0, 8);
};

// 选择项目
const handleSelectProject = (project: Project) => {
  selectedProject.value = project;
  selectedSession.value = null;
  messages.value = [];
  fetchSessions(project.path);
};

// 选择会话
const handleSelectSession = (session: Session) => {
  selectedSession.value = session;
  fetchSessionDetail(session.path);
};

// 处理最近会话点击
const handleRecentSessionClick = (sessionData: any) => {
  // 切换到项目视图
  currentView.value = 'projects';

  // 查找并选择项目
  const project = projects.value.find(p => p.path === sessionData.projectPath);
  if (project) {
    handleSelectProject(project);

    // 延迟获取会话列表后再选中会话
    setTimeout(() => {
      const session = sessions.value.find(s => s.id === sessionData.sessionId);
      if (session) {
        handleSelectSession(session);
      }
    }, 100);
  }
};

// 格式化时间
const formatDate = (date: Date) => {
  const d = new Date(date);
  return d.toLocaleDateString('zh-CN', { month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit' });
};

// 高亮搜索关键词
const highlightSearchTerm = (text: string, term: string) => {
  if (!term || !text) return text;
  const regex = new RegExp(`(${term.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')})`, 'gi');
  return text.replace(regex, '<mark class="search-highlight">$1</mark>');
};

// 组件挂载时获取项目列表
onMounted(() => {
  fetchProjects();
  fetchStats();
  // 添加键盘事件监听
  window.addEventListener('keydown', handleKeydown);
});

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown);
});
</script>

<template>
  <div class="app-container">
    <!-- 全局搜索弹窗 -->
    <ElDialog
      v-model="searchDialogVisible"
      title="全局搜索"
      width="600px"
      class="search-dialog"
      :show-close="true"
      @close="() => { searchQuery = ''; searchResults = []; }"
    >
      <ElInput
        v-model="searchQuery"
        placeholder="搜索对话内容... (Cmd+K)"
        class="search-input"
        clearable
        @input="performSearch"
      >
        <template #prefix>
          <span>🔍</span>
        </template>
      </ElInput>

      <div class="search-results">
        <div v-if="searching" class="search-loading">搜索中...</div>
        <div v-else-if="searchResults.length === 0 && searchQuery" class="search-empty">
          未找到匹配结果
        </div>
        <div v-else-if="!searchQuery" class="search-hint">
          输入关键词开始搜索，支持跨所有项目的对话内容
        </div>
        <div v-else class="search-list">
          <div
            v-for="(result, index) in searchResults"
            :key="index"
            class="search-result-item"
            @click="selectSearchResult(result)"
          >
            <div class="search-result-header">
              <span class="search-project">{{ result.projectName }}</span>
              <span class="search-session">{{ result.sessionId.slice(0, 8) }}</span>
            </div>
            <div class="search-preview">{{ result.preview }}</div>
          </div>
        </div>
      </div>
    </ElDialog>

    <!-- 快捷键帮助弹窗 -->
    <ElDialog
      v-model="shortcutsHelpVisible"
      title="快捷键帮助"
      width="500px"
      class="shortcuts-dialog"
      :show-close="true"
    >
      <div class="shortcuts-list">
        <div
          v-for="(shortcut, index) in shortcuts"
          :key="index"
          class="shortcut-item"
        >
          <div class="shortcut-key">
            <kbd>{{ shortcut.key }}</kbd>
          </div>
          <div class="shortcut-info">
            <div class="shortcut-desc">{{ shortcut.description }}</div>
            <div class="shortcut-action">{{ shortcut.action }}</div>
          </div>
        </div>
      </div>
      <div class="shortcuts-footer">
        <p>按 <kbd>?</kbd> 或 <kbd>Esc</kbd> 关闭</p>
      </div>
    </ElDialog>

    <div class="app-layout">
      <!-- 左侧图标导航栏 -->
      <nav class="nav-bar">
        <div class="logo">🤖</div>
        <button
          v-for="item in navItems"
          :key="item.key"
          class="nav-btn"
          :class="{ active: currentView === item.key }"
          :data-nav="item.key"
          :title="item.label"
          @click="currentView = item.key"
        >
          <svg v-if="item.key === 'dashboard'" width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <rect x="3" y="3" width="7" height="7" rx="1.5"/>
            <rect x="14" y="3" width="7" height="7" rx="1.5"/>
            <rect x="3" y="14" width="7" height="7" rx="1.5"/>
            <rect x="14" y="14" width="7" height="7" rx="1.5"/>
          </svg>
          <svg v-else-if="item.key === 'projects'" width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <path d="M3 7a2 2 0 012-2h4l2 2h8a2 2 0 012 2v9a2 2 0 01-2 2H5a2 2 0 01-2-2V7z"/>
          </svg>
          <svg v-else-if="item.key === 'search'" width="17" height="17" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.8">
            <circle cx="11" cy="11" r="7"/>
            <path d="M21 21l-4.35-4.35"/>
          </svg>
        </button>
      </nav>

      <!-- 主内容区 -->
      <div class="main-content">
      <!-- 项目列表面板 -->
      <aside v-if="currentView === 'projects'" class="project-panel">
        <div class="panel-title">
          <span>项目列表</span>
          <span class="badge">{{ projects.length }}</span>
        </div>
        <div v-if="loadingProjects" class="loading">加载中...</div>
        <div v-else-if="projects.length === 0" class="empty-text">暂无项目</div>
        <div v-else class="panel-list">
          <div
            v-for="project in projects"
            :key="project.path"
            class="project-item"
            :class="{ active: selectedProject?.path === project.path }"
            :data-project="project.path"
            @click="handleSelectProject(project)"
          >
            <span class="project-name">{{ project.name }}</span>
            <svg v-if="selectedProject?.path === project.path" class="chevron" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
              <path d="M9 18l6-6-6-6"/>
            </svg>
          </div>
        </div>
      </aside>

      <!-- 会话列表面板 -->
      <aside v-if="currentView === 'projects'" class="conv-panel">
        <div class="panel-title">
          <span>会话列表</span>
          <span class="badge">{{ sessions.length }}</span>
        </div>
        <div v-if="loadingSessions" class="loading">加载中...</div>
        <div v-else-if="!selectedProject" class="empty-text">请先选择项目</div>
        <div v-else-if="sessions.length === 0" class="empty-text">暂无会话</div>
        <div v-else class="panel-list">
          <div
            v-for="session in sessions"
            :key="session.id"
            class="conv-item"
            :class="{ active: selectedSession?.id === session.id }"
            :data-conv="session.id"
            @click="handleSelectSession(session)"
          >
            <div class="conv-title">{{ getSessionTitle(session.id) }}</div>
            <div class="conv-meta">
              <span class="conv-hash" :title="session.id">{{ session.id.slice(0, 8) }}</span>
              <span class="conv-time">{{ formatDate(session.modifiedAt) }}</span>
            </div>
          </div>
        </div>
      </aside>

      <!-- 搜索列表面板 -->
      <aside v-if="currentView === 'search'" class="search-panel">
        <div class="panel-title">全局搜索</div>
        <div class="search-view-content">
          <ElInput
            v-model="searchQuery"
            placeholder="搜索所有项目的对话内容..."
            size="large"
            class="search-view-input"
            clearable
            @input="performSearch"
          >
            <template #prefix>
              <span>🔍</span>
            </template>
          </ElInput>
          <div class="search-results-list">
            <div v-if="searching" class="search-loading">搜索中...</div>
            <div v-else-if="searchResults.length === 0 && searchQuery" class="search-empty">
              未找到匹配结果
            </div>
            <div v-else-if="!searchQuery" class="search-hint">
              输入关键词开始搜索，支持跨所有项目的对话内容
            </div>
            <div v-else class="search-result-items">
              <div
                v-for="(result, index) in searchResults"
                :key="index"
                class="search-result-item"
                :class="{ selected: selectedSearchResult?.sessionPath === result.sessionPath }"
                @click="selectSearchResult(result, false)"
                @dblclick="selectSearchResult(result, true)"
              >
                <div class="search-result-header">
                  <span class="search-project">{{ result.projectName }}</span>
                  <span class="search-session-id" :title="result.sessionId">{{ result.sessionId.slice(0, 8) }}</span>
                </div>
                <div class="search-preview" v-html="highlightSearchTerm(result.preview, searchQuery)"></div>
              </div>
            </div>
          </div>
        </div>
      </aside>

      <!-- 右侧主内容区 -->
      <main class="main-pane">
        <!-- 仪表盘视图 -->
        <div v-if="currentView === 'dashboard'" class="dashboard-main">
          <div v-if="loadingStats" class="dashboard-loading">加载统计中...</div>
          <div v-else-if="stats" class="dashboard-main-content">
            <!-- 活动统计 -->
            <ActivityStats :data="stats.contributionGraph || []" />

            <!-- 对话趋势图 -->
            <TrendChart :data="stats.contributionGraph || []" title="对话趋势" />

            <!-- GitHub 风格热力图 -->
            <ContributionGraph :data="stats.contributionGraph || []" />

            <!-- 项目分布饼图 -->
            <div class="dashboard-chart-section">
              <ProjectPieChart :data="stats.projectStats || []" />
            </div>

            <!-- 最近访问的会话 -->
            <RecentSessions
              v-if="stats.recentSessions && stats.recentSessions.length > 0"
              :data="stats.recentSessions"
              @click="handleRecentSessionClick"
            />
          </div>
          <div v-else class="dashboard-empty">
            <div class="empty-icon">📊</div>
            <p>暂无统计数据</p>
          </div>
        </div>

        <!-- 搜索预览窗格 -->
        <div v-if="currentView === 'search' && selectedSearchResult" class="preview-pane">
          <div class="preview-header">
            <div class="preview-info">
              <span class="preview-project">{{ selectedSearchResult.projectName }}</span>
              <span class="preview-session" :title="selectedSearchResult.sessionId">
                {{ selectedSearchResult.sessionId.slice(0, 12) }}
              </span>
            </div>
            <button class="preview-jump-btn" @click="selectSearchResult(selectedSearchResult, true)">
              在完整视图中打开 →
            </button>
          </div>

          <div v-if="previewLoading" class="preview-loading">加载中...</div>
          <div v-else-if="previewMessages.length === 0" class="preview-empty">
            <div class="empty-icon">📭</div>
            <p>暂无对话内容</p>
          </div>
          <div v-else class="preview-messages">
            <div
              v-for="(message, index) in previewMessages"
              :key="index"
              class="preview-message"
              :class="message.role"
            >
              <div class="preview-message-role">
                {{ message.role === 'user' ? 'You' : 'Claude' }}
              </div>
              <div class="preview-message-content">
                <MarkdownRenderer :content="message.content" />
              </div>
            </div>
          </div>
        </div>

        <!-- 搜索欢迎页 -->
        <div v-if="currentView === 'search' && !selectedSearchResult" class="welcome-screen">
          <div class="welcome-icon">🔍</div>
          <h1 class="welcome-title">全局搜索</h1>
          <p class="welcome-subtitle">在左侧输入关键词搜索对话</p>
          <p class="welcome-hint">单击搜索结果预览，双击跳转到完整视图</p>
        </div>

        <!-- 欢迎页 -->
        <div v-if="!selectedSession && currentView === 'projects'" class="welcome-screen">
          <div class="welcome-icon">💬</div>
          <h1 class="welcome-title">Claude 历史记录管理器</h1>
          <p class="welcome-subtitle">请从左侧选择项目和会话来查看对话详情</p>
          <div class="shortcut-hint">
            <span class="hint-icon">⌘</span>
            <span class="hint-key">K</span>
            <span class="hint-text">全局搜索</span>
          </div>
        </div>

        <!-- 对话详情 -->
        <div v-if="selectedSession && currentView === 'projects'" class="chat-view">
          <div class="chat-header">
            <div class="chat-info">
              <span class="chat-project">{{ selectedProject?.name }}</span>
              <span class="chat-session">{{ selectedSession?.id.slice(0, 12) }}</span>
            </div>
          </div>

          <div v-if="loadingMessages" class="loading">加载对话中...</div>
          <div v-else-if="messages.length === 0" class="empty-messages">
            <div class="empty-icon">📭</div>
            <p>暂无对话内容</p>
          </div>
          <div v-else class="messages-wrapper">
            <!-- 时间轴 -->
            <div class="timeline"></div>

            <div class="messages">
              <div
                v-for="(message, index) in messages"
                :key="index"
                class="message"
                :class="message.role"
              >
                <div class="message-timeline-point"></div>
                <div class="message-content">
                  <div class="message-header">
                    <span class="message-role">
                      {{ message.role === 'user' ? 'You' : 'Claude' }}
                    </span>
                    <span class="message-index">#{{ index + 1 }}</span>
                  </div>
                  <div class="message-body">
                    <MarkdownRenderer :content="message.content" />
                  </div>
                  <!-- 工具调用折叠面板 -->
                  <div v-if="message.toolCalls && message.toolCalls.length > 0" class="tool-calls-terminal">
                    <details class="terminal-panel">
                      <summary class="terminal-header">
                        <span class="terminal-prompt">$</span>
                        <span class="terminal-text">工具调用</span>
                        <span class="terminal-badge">{{ message.toolCalls.length }}</span>
                      </summary>
                      <div class="terminal-body">
                        <div
                          v-for="(toolCall, toolIndex) in message.toolCalls"
                          :key="toolIndex"
                          class="terminal-command"
                        >
                          <div class="command-header">
                            <span class="command-prompt">➜</span>
                            <code class="command-name">{{ toolCall.function.name }}</code>
                            <button
                              v-if="toolCall.function.name === 'Bash'"
                              class="copy-command-btn"
                              @click="copyToClipboard(toolCall.function.arguments, '命令')"
                              title="复制命令"
                            >
                              复制
                            </button>
                          </div>
                          <pre class="command-args">{{ toolCall.function.arguments }}</pre>
                        </div>
                      </div>
                    </details>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </main>
      </div>
    </div>
  </div>
</template>

<style scoped>
/* 应用布局 */
.app-layout {
  display: flex;
  flex-direction: row;
  height: 100vh;
  width: 100vw;
  background: #0d1117;
  font-family: 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', -apple-system, BlinkMacSystemFont, sans-serif;
  color: #c9d1d9;
  font-size: 13px;
  overflow: hidden;
}

/* 左侧图标导航栏 */
.nav-bar {
  width: 50px;
  background: #090c12;
  display: flex;
  flex-direction: column;
  align-items: center;
  padding-top: 12px;
  border-right: 1px solid #1c2333;
  flex-shrink: 0;
  gap: 2px;
}

.nav-bar .logo {
  width: 30px;
  height: 30px;
  border-radius: 8px;
  margin-bottom: 20px;
  background: linear-gradient(135deg, #f97316, #ef4444);
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 15px;
  flex-shrink: 0;
}

.nav-btn {
  width: 34px;
  height: 34px;
  border-radius: 7px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: all 0.15s;
  color: #4b5563;
  background: transparent;
  border: none;
  outline: none;
  box-shadow: none;
  flex-shrink: 0;
}

.nav-btn:hover {
  color: #9ca3af;
  background: #0f1520;
}

.nav-btn.active {
  color: #93c5fd;
  background: #162032;
  box-shadow: inset 2px 0 0 #3b82f6;
}

/* 项目列表面板 */
.project-panel {
  width: 155px;
  background: #0d1117;
  border-right: 1px solid #1c2333;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  overflow: hidden;
}

/* 会话列表面板 */
.conv-panel {
  width: 195px;
  background: #0b0f18;
  border-right: 1px solid #1c2333;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  overflow: hidden;
}

/* 搜索列表面板 */
.search-panel {
  width: 350px;
  background: #0b0f18;
  border-right: 1px solid #1c2333;
  display: flex;
  flex-direction: column;
  flex-shrink: 0;
  overflow: hidden;
}

.panel-title {
  padding: 14px 12px 8px;
  font-size: 10px;
  font-weight: 700;
  color: #3d4f63;
  letter-spacing: 0.1em;
  text-transform: uppercase;
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-shrink: 0;
}

.badge {
  background: #1c2333;
  color: #4b5563;
  font-size: 10px;
  padding: 1px 6px;
  border-radius: 10px;
}

.panel-list {
  flex: 1;
  overflow-y: auto;
}

/* 项目行 */
.project-item {
  padding: 7px 12px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: space-between;
  color: #6b7280;
  border-left: 2px solid transparent;
  transition: all 0.12s;
  white-space: nowrap;
  overflow: hidden;
}

.project-item:hover {
  color: #9ca3af;
  background: #0f1520;
}

.project-item.active {
  color: #e2e8f0;
  background: #13192a;
  border-left-color: #3b82f6;
  font-weight: 500;
}

.project-name {
  overflow: hidden;
  text-overflow: ellipsis;
  font-size: 12.5px;
}

.chevron {
  flex-shrink: 0;
  opacity: 0.5;
}

/* 会话行 */
.conv-item {
  padding: 9px 12px;
  cursor: pointer;
  transition: all 0.12s;
  border-left: 2px solid transparent;
}

.conv-item:hover {
  background: #0f1520;
}

.conv-item.active {
  background: #13192a;
  border-left-color: #3b82f6;
}

.conv-title {
  font-size: 12.5px;
  color: #9ca3af;
  margin-bottom: 4px;
}

.conv-item.active .conv-title {
  color: #e2e8f0;
  font-weight: 500;
}

.conv-meta {
  display: flex;
  align-items: center;
  gap: 5px;
}

.conv-hash {
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 9.5px;
  background: #1c2333;
  color: #4b5563;
  padding: 1px 5px;
  border-radius: 3px;
}

.conv-time {
  font-size: 10.5px;
  color: #374151;
}

/* 搜索视图内容 */
.search-view-content {
  flex: 1;
  display: flex;
  flex-direction: column;
  padding: 12px;
  overflow: hidden;
}

.search-view-input {
  flex-shrink: 0;
}

.search-results-list {
  flex: 1;
  overflow-y: auto;
  margin-top: 12px;
}

/* 主内容区 */
.main-content {
  flex: 1;
  display: flex;
  overflow: hidden;
  min-height: 0;
}

.loading,
.empty-text {
  padding: 20px;
  text-align: center;
  color: #374151;
  font-size: 13px;
}

/* 视图容器 */
.pane-view {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.pane-section {
  display: flex;
  flex-direction: column;
  flex: 1;
  min-height: 0;
  overflow: hidden;
}

.projects-section,
.sessions-section {
  flex: 1;
  overflow: hidden;
  min-height: 0;
}

.pane-header {
  padding: 16px;
  border-bottom: 1px solid var(--border-color);
}

.pane-title {
  margin: 0;
  font-size: 15px;
  font-weight: 600;
  color: var(--text-primary);
}

.section-title {
  margin: 0;
  font-size: 12px;
  font-weight: 600;
  color: var(--text-secondary);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.pane-content {
  flex: 1;
  overflow-y: auto;
  padding: 16px;
  min-height: 0;
}

.list-container {
  flex: 1;
  overflow-y: auto;
  padding: 8px;
  min-height: 0;
}

.list-item {
  padding: 10px 12px;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.15s ease;
  margin-bottom: 2px;
  color: var(--text-secondary);
  font-size: 13px;
}

.list-item:hover {
  background: var(--hover-bg);
  color: var(--text-primary);
}

.list-item.active {
  background: var(--accent-bg);
  color: var(--accent-color);
}

.item-name {
  font-size: 13px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.session-title {
  font-size: 13px;
  font-weight: 500;
  color: var(--text-primary);
  margin-bottom: 4px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.session-meta {
  display: flex;
  align-items: center;
  gap: 6px;
}

.session-id {
  font-size: 10px;
  font-family: var(--font-mono);
  color: var(--text-muted);
  background: var(--bg-tertiary);
  padding: 2px 5px;
  border-radius: 3px;
}

.session-time {
  font-size: 10px;
  color: var(--text-muted);
}

.loading,
.empty-text {
  padding: 20px;
  text-align: center;
  color: var(--text-muted);
  font-size: 13px;
}

/* 右侧主内容区 */
.main-pane {
  flex: 1;
  background: #0d1117;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

/* 仪表盘主区域 */
.dashboard-main {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow-y: auto;
  padding: 24px 32px;
  gap: 20px;
  max-width: 1200px;
  margin: 0 auto;
  width: 100%;
}

.dashboard-loading,
.dashboard-empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #374151;
  min-height: 400px;
}

.dashboard-empty .empty-icon {
  font-size: 64px;
  margin-bottom: 16px;
  opacity: 0.5;
}

.dashboard-main-content {
  display: flex;
  flex-direction: column;
  gap: 20px;
}

.dashboard-chart-section {
  background: #13192a;
  border-radius: 10px;
  border: 1px solid #1c2333;
  padding: 16px;
}

/* 欢迎屏幕 */
.welcome-screen {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #374151;
  overflow: hidden;
  min-height: 0;
}

.welcome-icon {
  font-size: 64px;
  margin-bottom: 20px;
  opacity: 0.6;
}

.welcome-title {
  margin: 0 0 8px 0;
  font-size: 24px;
  font-weight: 600;
  color: #9ca3af;
}

.welcome-subtitle {
  margin: 0 0 24px 0;
  font-size: 13px;
  color: #4b5563;
}

.shortcut-hint {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px 16px;
  background: #13192a;
  border: 1px solid #1c2333;
  border-radius: 6px;
  font-size: 12px;
  color: #6b7280;
}

.hint-icon {
  font-size: 14px;
  color: #4b5563;
}

.hint-key {
  padding: 3px 7px;
  background: #1c2333;
  border: 1px solid #1c2333;
  border-radius: 4px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 11px;
  font-weight: 600;
}

.hint-text {
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.welcome-hint {
  margin-top: 12px;
  font-size: 13px;
  color: #4b5563;
}

/* 对话视图 */
.chat-view {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.chat-header {
  height: 44px;
  border-bottom: 1px solid #1c2333;
  display: flex;
  align-items: center;
  padding: 0 20px;
  gap: 8px;
  background: #0d1117;
  flex-shrink: 0;
}

.chat-info {
  display: flex;
  align-items: center;
  gap: 8px;
}

.chat-project {
  font-size: 13.5px;
  font-weight: 600;
  color: #e2e8f0;
}

.chat-session {
  background: #1c2333;
  color: #6b7280;
  font-size: 11px;
  padding: 2px 8px;
  border-radius: 4px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
}

.empty-messages {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #374151;
  overflow: hidden;
  min-height: 0;
}

.empty-icon {
  font-size: 48px;
  margin-bottom: 12px;
}

.empty-messages p {
  font-size: 13px;
}

/* 消息列表 */
.messages-wrapper {
  flex: 1;
  display: flex;
  position: relative;
  overflow: hidden;
  min-height: 0;
}

.messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px 24px;
  min-height: 0;
}

.message {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
}

.message-timeline-point {
  width: 7px;
  height: 7px;
  border-radius: 50%;
  flex-shrink: 0;
  margin-top: 9px;
}

.message.user .message-timeline-point {
  background: #3b82f6;
}

.message.assistant .message-timeline-point {
  background: #10b981;
}

.message-content {
  flex: 1;
  min-width: 0;
}

.message-header {
  display: flex;
  align-items: baseline;
  gap: 7px;
  margin-bottom: 6px;
}

.message-role {
  font-size: 12.5px;
  font-weight: 600;
}

.message.user .message-role {
  color: #60a5fa;
}

.message.assistant .message-role {
  color: #34d399;
}

.message-index {
  font-size: 11px;
  color: #2d3748;
}

.message-body {
  font-size: 13.5px;
  color: #c9d1d9;
  line-height: 1.7;
}

.message.user .message-body {
  background: #0e1828;
  padding: 10px 14px;
  border-radius: 8px;
  border-left: 2px solid #1e3a5f;
}

/* 终端风格工具调用面板 */
.tool-calls-terminal {
  margin-top: 8px;
}

.terminal-panel {
  background: #0d1117;
  border: 1px solid #1c2333;
  border-radius: 7px;
  overflow: hidden;
}

.terminal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 6px;
  padding: 8px 12px;
  cursor: pointer;
  user-select: none;
  background: transparent;
  transition: background 0.15s ease;
  list-style: none;
}

.terminal-header::-webkit-details-marker {
  display: none;
}

.terminal-header:hover {
  background: #0f1520;
}

.terminal-panel[open] .terminal-header {
  background: #0f1520;
}

.terminal-prompt {
  color: #4b5563;
  font-size: 12px;
}

.terminal-text {
  color: #4b5563;
  font-size: 12px;
}

.terminal-badge {
  background: #1c2333;
  color: #4b5563;
  font-size: 10px;
  width: 18px;
  height: 18px;
  border-radius: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-left: auto;
}

.terminal-body {
  padding: 8px 12px;
  max-height: 300px;
  overflow-y: auto;
}

.terminal-command {
  margin-bottom: 8px;
}

.terminal-command:last-child {
  margin-bottom: 0;
}

.command-header {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 6px;
}

.command-prompt {
  color: #4b5563;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 11px;
}

.command-name {
  color: #2d3748;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 11px;
}

.copy-command-btn {
  margin-left: auto;
  padding: 2px 8px;
  background: #1c2333;
  border: 1px solid #1c2333;
  border-radius: 3px;
  color: #6b7280;
  font-size: 10px;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: 'PingFang SC', 'Hiragino Sans GB', 'Microsoft YaHei', sans-serif;
}

.copy-command-btn:hover {
  background: #2d3748;
  color: #9ca3af;
}

.copy-command-btn:active {
  transform: scale(0.95);
}

.command-args {
  margin: 0;
  padding: 8px;
  background: #1c2333;
  border-radius: 4px;
  font-size: 11px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  white-space: pre-wrap;
  word-break: break-word;
  max-height: 150px;
  overflow-y: auto;
  color: #c9d1d9;
}

/* 仪表盘样式 */
.dashboard-content {
  flex: 1;
  overflow-y: auto;
  min-height: 0;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 24px;
}

.stat-card {
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 12px;
  padding: 20px;
  text-align: center;
  transition: all 0.2s ease;
}

.stat-card:hover {
  border-color: var(--accent-color);
}

.stat-icon {
  font-size: 28px;
  margin-bottom: 8px;
}

.stat-value {
  font-size: 32px;
  font-weight: 700;
  color: var(--text-primary);
  line-height: 1;
  margin-bottom: 4px;
}

.stat-label {
  font-size: 12px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.project-stats-section {
  margin-top: 24px;
}

.project-stats-section .section-title {
  margin-bottom: 16px;
  padding: 0;
}

.project-stats-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.project-stat-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  background: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 8px;
}

.project-stat-info {
  display: flex;
  align-items: center;
  gap: 10px;
  min-width: 180px;
}

.project-stat-rank {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--accent-color);
  color: #000;
  border-radius: 50%;
  font-size: 11px;
  font-weight: 700;
}

.project-stat-name {
  flex: 1;
  font-size: 13px;
  color: var(--text-primary);
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.project-stat-bar {
  flex: 1;
  display: flex;
  align-items: center;
  gap: 10px;
  height: 20px;
}

.project-stat-bar-fill {
  height: 6px;
  background: linear-gradient(90deg, var(--accent-color), var(--accent-hover));
  border-radius: 3px;
  min-width: 0;
}

.project-stat-count {
  font-size: 12px;
  color: var(--text-secondary);
  font-weight: 600;
  white-space: nowrap;
}

/* 搜索视图样式 */
.search-view-input {
  margin-bottom: 16px;
}

.search-results-list {
  flex: 1;
  overflow-y: auto;
  min-height: 0;
}

.search-result-items {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

/* 全局搜索弹窗样式 */
.search-dialog :deep(.el-dialog) {
  background: #0d1117;
  border: 1px solid #1c2333;
}

.search-dialog :deep(.el-dialog__header) {
  border-bottom: 1px solid #1c2333;
}

.search-dialog :deep(.el-dialog__title) {
  color: #c9d1d9;
}

.search-dialog :deep(.el-dialog__body) {
  padding: 12px;
}

.search-input :deep(.el-input__wrapper) {
  background: #0d1117;
  box-shadow: none;
  border: 1px solid #1c2333;
}

.search-input :deep(.el-input__wrapper:hover) {
  border-color: #30363d;
}

.search-input :deep(.el-input__wrapper.is-focus) {
  border-color: #3b82f6;
}

.search-input :deep(.el-input__inner) {
  color: #c9d1d9;
  background: transparent;
}

.search-input :deep(.el-input__inner::placeholder) {
  color: #4b5563;
}

.search-input :deep(.el-input__prefix) {
  display: flex;
  align-items: center;
  justify-content: center;
}

.search-input :deep(.el-input__prefix span) {
  font-size: 16px;
  opacity: 0.6;
}

.search-results {
  max-height: 400px;
  overflow-y: auto;
}

.search-loading,
.search-empty,
.search-hint {
  padding: 20px;
  text-align: center;
  color: #374151;
  font-size: 13px;
}

.search-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.search-result-item {
  padding: 8px 12px;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.12s ease;
  background: transparent;
}

.search-result-item:hover {
  background: #0f1520;
}

.search-result-header {
  display: flex;
  align-items: center;
  gap: 6px;
  margin-bottom: 4px;
  font-size: 11px;
}

.search-project {
  font-weight: 500;
  color: #9ca3af;
}

.search-session {
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  color: #4b5563;
  background: #1c2333;
  padding: 1px 5px;
  border-radius: 3px;
  font-size: 9.5px;
}

.search-preview {
  font-size: 12px;
  color: #6b7280;
  line-height: 1.5;
  overflow: hidden;
  text-overflow: ellipsis;
  display: -webkit-box;
  -webkit-line-clamp: 2;
  -webkit-box-orient: vertical;
}

.search-result-item.selected {
  background: #13192a;
  border-left: 2px solid #3b82f6;
}

.search-result-item.selected .search-preview {
  color: #9ca3af;
}

/* 快捷键帮助弹窗样式 */
.shortcuts-dialog :deep(.el-dialog) {
  background: #0d1117;
  border: 1px solid #1c2333;
}

.shortcuts-dialog :deep(.el-dialog__header) {
  border-bottom: 1px solid #1c2333;
}

.shortcuts-dialog :deep(.el-dialog__title) {
  color: #c9d1d9;
}

.shortcuts-dialog :deep(.el-dialog__body) {
  padding: 0;
}

.shortcuts-list {
  display: flex;
  flex-direction: column;
}

.shortcut-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-bottom: 1px solid #1c2333;
  transition: background 0.15s ease;
}

.shortcut-item:hover {
  background: #0f1520;
}

.shortcut-item:last-child {
  border-bottom: none;
}

.shortcut-key {
  flex-shrink: 0;
}

.shortcut-key kbd {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  padding: 4px 8px;
  background: #1c2333;
  border: 1px solid #1c2333;
  border-radius: 4px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 11px;
  font-weight: 600;
  color: #93c5fd;
  min-width: 50px;
}

.shortcut-info {
  flex: 1;
}

.shortcut-desc {
  font-size: 13px;
  font-weight: 500;
  color: #9ca3af;
  margin-bottom: 2px;
}

.shortcut-action {
  font-size: 11px;
  color: #4b5563;
}

.shortcuts-footer {
  padding: 12px 16px;
  background: #13192a;
  border-top: 1px solid #1c2333;
  text-align: center;
}

.shortcuts-footer p {
  margin: 0;
  font-size: 11px;
  color: #4b5563;
}

.shortcuts-footer kbd {
  padding: 3px 6px;
  background: #1c2333;
  border: 1px solid #1c2333;
  border-radius: 3px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  font-size: 10px;
  margin: 0 2px;
}

.search-highlight {
  background: rgba(59, 130, 246, 0.3);
  color: #c9d1d9;
  padding: 0 2px;
  border-radius: 2px;
  font-weight: 500;
}

.search-session-id {
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  color: #4b5563;
  background: #1c2333;
  padding: 1px 5px;
  border-radius: 3px;
  font-size: 9.5px;
}

/* 搜索预览窗格 */
.preview-pane {
  flex: 1;
  display: flex;
  flex-direction: column;
  overflow: hidden;
  min-height: 0;
}

.preview-header {
  padding: 12px 20px;
  border-bottom: 1px solid #1c2333;
  background: #0d1117;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.preview-info {
  display: flex;
  align-items: center;
  gap: 8px;
}

.preview-project {
  font-size: 13px;
  font-weight: 600;
  color: #e2e8f0;
}

.preview-session {
  font-size: 11px;
  font-family: 'JetBrains Mono', 'Fira Code', 'SF Mono', Consolas, monospace;
  color: #6b7280;
  padding: 2px 8px;
  background: #1c2333;
  border-radius: 4px;
}

.preview-jump-btn {
  padding: 6px 12px;
  background: #3b82f6;
  color: #e2e8f0;
  border: none;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s ease;
}

.preview-jump-btn:hover {
  background: #60a5fa;
}

.preview-loading,
.preview-empty {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: #374151;
  overflow: hidden;
  min-height: 0;
}

.preview-messages {
  flex: 1;
  overflow-y: auto;
  padding: 20px 24px;
  min-height: 0;
}

.preview-message {
  margin-bottom: 20px;
  display: flex;
  gap: 10px;
}

.preview-message.user .preview-message-role {
  color: #60a5fa;
}

.preview-message.assistant .preview-message-role {
  color: #34d399;
}

.preview-message-role {
  font-size: 12.5px;
  font-weight: 600;
  color: #9ca3af;
  flex-shrink: 0;
  width: 40px;
  margin-top: 9px;
}

.preview-message-content {
  color: #c9d1d9;
  line-height: 1.6;
  font-size: 13px;
  flex: 1;
  min-width: 0;
}

.preview-message-content :deep(p) {
  margin: 0.5em 0;
}

.preview-message-content :deep(code) {
  background: var(--bg-tertiary);
  padding: 0.2em 0.4em;
  border-radius: 4px;
  font-size: 0.85em;
}

.preview-message-content :deep(pre) {
  background: var(--bg-primary);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  padding: 12px;
  overflow-x: auto;
}
</style>

<style>
/* 全局样式重置 */
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, 'Helvetica Neue', Arial, sans-serif;
  font-size: 14px;
  line-height: 1.5;
  color: var(--text-primary);
}

#app {
  height: 100vh;
  width: 100vw;
}
</style>
