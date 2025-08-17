<template>
  <div class="container">
    <h1><el-icon><ChatDotRound /></el-icon> 某种聊天记录</h1>
    <div class="file-input-container">
      <label for="fileInput" class="file-input-label">
        <el-icon><Upload /></el-icon>
        选择 JSON 文件
      </label>
      <input type="file" id="fileInput" accept=".json" @change="handleFileChange">
      <label for="txtFileInput" class="file-input-label" style="margin-left: 12px;">
        <el-icon><Upload /></el-icon>
        上传聊天txt
      </label>
      <input type="file" id="txtFileInput" accept=".txt" multiple @change="handleTxtFileChange">
    </div>
    
    <!-- 文件列表 -->
    <div class="file-list">
      <h2><el-icon><Document /></el-icon> 聊天窗口列表</h2>
      <div class="search-container">
        <el-input
          v-model="searchQuery"
          placeholder="搜索聊天记录..."
          :prefix-icon="Search"
          clearable
          @input="handleSearch"
        >
          <template #prefix>
            <el-icon><Search /></el-icon>
          </template>
        </el-input>
      </div>
      <div class="file-list-content">
        <div v-if="lastOutputs.length > 0" class="select-all-container">
          <label class="select-all-label">
            <input 
              type="checkbox" 
              :checked="isAllSelected" 
              @change="toggleSelectAll"
            >
            <el-icon><Select /></el-icon>
            选择所有
          </label>
        </div>
        <ul v-if="lastOutputs.length > 0">
          <li v-for="(output, index) in filteredOutputs" 
              :key="index" 
              class="file-item">
            <label class="file-checkbox">
              <input 
                type="checkbox" 
                v-model="selectedFiles[index]"
                @change="updateSelection"
              >
            </label>
            <span class="file-name" @click="showFileContent(output)">
              <el-icon><ChatLineRound /></el-icon>
              {{ output.filename }}
            </span>
          </li>
        </ul>
        <div v-else class="empty-state">
          <el-icon><UploadFilled /></el-icon>
          请上传 conversation.json 文件 以查看聊天记录
        </div>
      </div>
    </div>

    <!-- 弹窗 -->
    <div class="modal" v-if="showModal" @click.self="closeModal">
      <div class="modal-content">
        <div class="modal-header">
          <div class="header-top">
          <h3><el-icon><Document /></el-icon> {{ currentFile?.filename }}</h3>
            <div class="header-actions">
              <div class="statistics-note">
                <el-tooltip
                  effect="dark"
                  content="精确计算采用GPT官方分词器tokenizer-GPT 4o，不适用于Claude。考虑到聊天内容还包含文件、图片等无法被提取的内容，token计算结果仅供参考。"
                  placement="bottom"
                >
                  <el-icon><InfoFilled /></el-icon>
                </el-tooltip>
              </div>
          <button class="close-button" @click="closeModal">
            <el-icon><Close /></el-icon>
          </button>
            </div>
          </div>
          <div class="statistics">
            <div class="stat-item" style="width: 120px">
              <span class="stat-label">总字数：</span>
              <span class="stat-value">{{ statistics.totalTokens }}</span>
            </div>
            <div class="stat-item" style="width: 120px">
              <span class="stat-label">AI回复：</span>
              <span class="stat-value">{{ statistics.assistantCount }}次</span>
            </div>
            <div class="stat-item" style="width: 120px">
              <span class="stat-label">用户消息：</span>
              <span class="stat-value">{{ statistics.userCount }}次</span>
            </div>
            <div class="export-current">
              <button 
                class="mode-toggle-button" 
                @click="toggleSelectMode"
                :title="isSelectMode ? '退出选择模式' : '进入选择模式'"
              >
                <el-icon><Select /></el-icon>
                {{ isSelectMode ? '退出选择' : '选择消息' }}
              </button>
              <template v-if="isSelectMode">
                <button 
                  class="select-all-button" 
                  @click="toggleSelectAllMessages"
                  :title="isAllMessagesSelected ? '取消全选' : '全选消息'"
                >
                  <el-icon><Select /></el-icon>
                  {{ isAllMessagesSelected ? '取消全选' : '全选' }}
                </button>
                <button 
                  class="export-button" 
                  @click="exportSelectedMessages"
                  :disabled="!hasSelectedMessages"
                  title="导出选中的消息"
                >
                  <el-icon><Download /></el-icon>
                  导出选中 ({{ selectedMessagesCount }})
                </button>
              </template>
            </div>
            <div v-if="!isSelectMode" class="token-calc">
              <button 
                class="token-button" 
                @click="calculateExactTokens"
                :disabled="isCalculating"
              >
                <el-icon v-if="!isCalculating"><InfoFilled /></el-icon>
                <el-icon v-else class="is-loading"><Loading /></el-icon>
                {{ isCalculating ? '计算中...' : '计算精确Token' }}
              </button>
              <span v-if="exactTokens !== null" class="stat-value">
                <span class="token-value">{{ exactTokens }}</span>
                <span class="token-label">tokens</span>
              </span>
            </div>
          </div>
          <!-- 搜索高亮导航 -->
          <div v-if="searchKeyword && matchCount > 0" class="search-nav">
            <button @click="gotoPrevMatch" :disabled="matchCount <= 1">上一个</button>
            <span>{{ currentMatchIndex + 1 }} / {{ matchCount }}</span>
            <button @click="gotoNextMatch" :disabled="matchCount <= 1">下一个</button>
          </div>
        </div>
        <div class="modal-body">
          <div class="chat-container">
            <template v-for="(line, index) in chatLines" :key="index">
              <div class="message-container" :class="{ 'selected': isSelectMode && selectedMessages[index] }">
                <div v-if="isSelectMode" class="message-checkbox">
                  <input 
                    type="checkbox" 
                    v-model="selectedMessages[index]"
                    :id="`msg-${index}`"
                  />
                  <label :for="`msg-${index}`"></label>
                </div>
                <div :class="['message-wrapper', getMessageClass(line)]">
                  <div class="avatar">
                    <el-icon v-if="isLeftMessage(line)" ><ChatDotRound /></el-icon>
                    <el-icon v-else><User /></el-icon>
                  </div>
                  <div class="message-bubble" v-html="getMessageContent(line, index)" />
                </div>
              </div>
            </template>
          </div>
        </div>
      </div>
    </div>

    <div class="button-container">
      <button @click="handleDownload" :disabled="!canDownload">
        <el-icon><Download /></el-icon>
        导出选中对话
      </button>
    </div>
    <div class="status" v-text="status"></div>
  </div>
</template>

<script>
import { ref, computed, onMounted, nextTick, watch } from 'vue'
import JSZip from 'jszip'
import { marked } from 'marked'
import { ElMessage } from 'element-plus'
import { 
  ChatDotRound, 
  Upload, 
  Document, 
  Select, 
  ChatLineRound, 
  UploadFilled, 
  Close, 
  Download,
  User,
  Search,
  InfoFilled,
  Loading
} from '@element-plus/icons-vue'
import { preloadEncoder, calculateTokens } from '@/utils/tokenizerLoader'

// 配置 marked 允许 HTML 标签
marked.setOptions({
  breaks: true,
  gfm: true,
  headerIds: false,
  mangle: false,
  pedantic: false,
  smartLists: true,
  smartypants: false
});

export default {
  name: 'App',
  components: {
    ChatDotRound,
    Upload,
    Document,
    Select,
    ChatLineRound,
    UploadFilled,
    Close,
    Download,
    User,
    Search,
    InfoFilled,
    Loading
  },
  setup() {
    const outputContent = ref('聊天记录将在这里显示');
    const status = ref('');
    const lastOutputs = ref([]);
    const selectedFiles = ref([]);
    const searchQuery = ref('');
    const filteredOutputs = ref([]);
    const canDownload = computed(() => selectedFiles.value.some(selected => selected));
    const showModal = ref(false);
    const currentFile = ref(null);
    const isCalculating = ref(false)
    const exactTokens = ref(null)
    const searchKeyword = ref('')
    const matchCount = ref(0)
    const currentMatchIndex = ref(0)
    const matchRefs = ref([])
    
    // 新增：用于跟踪选中的消息
    const selectedMessages = ref([])
    const isSelectMode = ref(false)

    const isAllSelected = computed(() => {
      return lastOutputs.value.length > 0 && 
             selectedFiles.value.every(selected => selected);
    });

    // 新增：计算属性 - 是否所有消息都被选中
    const isAllMessagesSelected = computed(() => {
      return chatLines.value.length > 0 && 
             selectedMessages.value.length === chatLines.value.length &&
             selectedMessages.value.every(selected => selected);
    });

    // 新增：计算属性 - 选中消息的数量
    const selectedMessagesCount = computed(() => {
      return selectedMessages.value.filter(selected => selected).length;
    });

    // 新增：计算属性 - 是否有选中的消息
    const hasSelectedMessages = computed(() => {
      return selectedMessagesCount.value > 0;
    });

    function toggleSelectAll(event) {
      const checked = event.target.checked;
      selectedFiles.value = lastOutputs.value.map(() => checked);
    }

    function updateSelection() {
      // 当手动选择文件时，更新选择状态
      selectedFiles.value = [...selectedFiles.value];
    }

    function sanitizeFilename(filename) {
      return filename.replace(/[\\/*?:"<>|]/g, "");
    }

    function extractDialogueFromMapping(mapping) {
      const dialogues = [];
      // 添加空值检查
      if (!mapping || typeof mapping !== 'object') {
        return dialogues;
      }
      
      for (const randomKey in mapping) {
        try {
          const item = mapping[randomKey];
          if (!item || !item.message) continue;
          
          const msg = item.message;
          const role = msg.author?.role || "unknown";
          if (!["user", "assistant"].includes(role)) continue;
          const content = msg.content || {};
          let timestamp = '';
          
          // 处理时间戳
          if (msg.create_time) {
            timestamp = formatTimestamp(msg.create_time);
          } else if (msg.created_at) {
            timestamp = formatTimestamp(msg.created_at);
          }
          
          if (content.parts && content.parts.length) {
            const text = content.parts.join(" ").replace(/^"|"$/g, "").trim();
            if (text) {
              dialogues.push(`【${role}】${timestamp ? ` [${timestamp}]` : ''} ${text}`);
            }
          } else if (role === "user") {
            const metadata = msg.metadata || {};
            const userData = metadata.user_context_message_data || {};
            const aboutModel = (userData.about_model_message || "").trim();
            const aboutUser = (userData.about_user_message || "").trim();
            let added = false;
            if (aboutModel) {
              dialogues.push(`【model_message】${timestamp ? ` [${timestamp}]` : ''} ${aboutModel}`);
              added = true;
            }
            if (aboutUser) {
              dialogues.push(`【user_message】${timestamp ? ` [${timestamp}]` : ''} ${aboutUser}`);
              added = true;
            }
            if (added) {
              dialogues.push("-".repeat(80));
            }
          }
        } catch (e) {
          console.log(`处理 mapping 内随机键 ${randomKey} 时出错：${e}`);
        }
      }
      return dialogues;
    }

    function extractDialogueFromDeepSeekMapping(mapping) {
      const dialogues = [];
      const messages = [];
      
      // 添加空值检查
      if (!mapping || typeof mapping !== 'object') {
        return dialogues;
      }
      
      // 首先收集所有消息
      for (const key in mapping) {
          const item = mapping[key];
          if (item && item.message) {
            const msg = item.message;
            
            // 检查新格式：fragments 下的 content 和 type
            if (msg.fragments && Array.isArray(msg.fragments)) {
              for (const fragment of msg.fragments) {
                if (fragment.content && fragment.type) {
                  let role = '';
                  if (fragment.type === 'RESPONSE') {
                    role = 'assistant';
                  } else if (fragment.type === 'REQUEST') {
                    role = 'user';
                  } else {
                    continue; // 跳过未知类型
                  }
                  
                  messages.push({
                    id: item.id,
                    parent: item.parent,
                    content: fragment.content,
                    role: role,
                    inserted_at: msg.inserted_at
                  });
                }
              }
            }
            // 兼容旧格式：直接在 message.content 中
            else if (msg.content && typeof msg.content === 'string') {
              messages.push({
                id: item.id,
                parent: item.parent,
                content: msg.content,
                inserted_at: msg.inserted_at
              });
            }
          }
      }
      
      // 按时间排序
      messages.sort((a, b) => {
        if (a.inserted_at && b.inserted_at) {
          return new Date(a.inserted_at) - new Date(b.inserted_at);
        }
        return 0;
      });
      
      // 先构建id到消息的映射（仅用于旧格式的角色判断）
      const idMap = {};
      messages.forEach(msg => { idMap[msg.id] = msg; });

      // 构建对话
      for (const msg of messages) {
        let role = msg.role || ''; // 新格式已经有角色信息
        let timestamp = '';
        
        // 如果没有角色信息（旧格式），则通过父子关系判断
        if (!role) {
          if (msg.parent === 'root') {
            role = 'user';
          } else {
            const parentMsg = idMap[msg.parent];
            if (parentMsg) {
              // 直接用父消息的角色反转
              role = parentMsg.role === 'user' ? 'assistant' : 'user';
            } else {
              role = 'assistant'; // fallback
            }
          }
          msg.role = role; // 记录下来，供后续子消息判断
        }
        
        // 处理时间戳
        if (msg.inserted_at) {
          timestamp = formatTimestamp(msg.inserted_at);
        }
        
        // 处理内容
        if (msg.content && typeof msg.content === 'string') {
          const text = msg.content.trim();
          if (text) {
            const dialogue = `【${role}】${timestamp ? ` [${timestamp}]` : ''} ${text}`;
            dialogues.push(dialogue);
          }
        }
      }
      
      return dialogues;
    }

    // 添加时间戳转换函数
    function formatTimestamp(timestamp) {
      try {
        let date;
        if (typeof timestamp === 'number') {
          // 处理Unix时间戳
          date = new Date(timestamp * 1000);
        } else if (typeof timestamp === 'string') {
          // 处理ISO格式时间戳
          date = new Date(timestamp);
        } else {
          return '';
        }
        
        if (isNaN(date.getTime())) {
          return '';
        }
        
        // 转换为东八区时间
        const utcDate = new Date(date.getTime() + (date.getTimezoneOffset() * 60000));
        const beijingDate = new Date(utcDate.getTime() + (8 * 60 * 60 * 1000));
        
        const year = beijingDate.getFullYear();
        const month = String(beijingDate.getMonth() + 1).padStart(2, '0');
        const day = String(beijingDate.getDate()).padStart(2, '0');
        const hours = String(beijingDate.getHours()).padStart(2, '0');
        const minutes = String(beijingDate.getMinutes()).padStart(2, '0');
        const seconds = String(beijingDate.getSeconds()).padStart(2, '0');
        
        return `${year}-${month}-${day} ${hours}:${minutes}:${seconds}`;
      } catch (e) {
        console.error('时间戳转换错误:', e);
        return '';
      }
    }

    function processRawJson(data) {
      const outputs = [];
      
      // 检查是否为数组格式
      if (Array.isArray(data)) {
        const counter = {};
        for (const item of data) {
          // 检查是否为Claude格式
          if (item.name !== undefined && item.chat_messages !== undefined) {
            const name = item.name || "Undefined";
            const chatMessages = item.chat_messages || [];
            
            if (!chatMessages.length) {
              continue;
            }
            
            const dialogues = [];
            for (const msg of chatMessages) {
              const sender = msg.sender;
              const text = msg.text;
              const timestamp = msg.created_at ? formatTimestamp(msg.created_at) : '';
              if (sender && text) {
                const role = sender === "human" ? "user" : "assistant";
                dialogues.push(`【${role}】${timestamp ? ` [${timestamp}]` : ''} ${text}`);
              }
            }
            
            if (dialogues.length) {
              const base = sanitizeFilename(name);
              const count = (counter[base] || 0) + 1;
              counter[base] = count;
              const fn = count > 1 ? `${base}_${count}.txt` : `${base}.txt`;
              outputs.push({ filename: fn, content: dialogues.join("\n") });
            }
          }
          // 检查是否为DeepSeek格式（优先于ChatGPT）
          else if (item.title !== undefined && item.mapping !== undefined) {
            // 进一步验证是否为 DeepSeek 格式
            const mapping = item.mapping;
            // 添加空值检查
            if (!mapping || typeof mapping !== 'object') continue;
            let isDeepSeek = false;
            
            // 检查 mapping 中是否有 DeepSeek 特有的结构
            for (const key in mapping) {
              if (mapping[key] && mapping[key].message) {
                const msg = mapping[key].message;
                // 检查新格式：fragments 结构
                if (msg.fragments && Array.isArray(msg.fragments) && 
                    msg.fragments.some(f => f.content && f.type && 
                    (f.type === 'RESPONSE' || f.type === 'REQUEST'))) {
                  isDeepSeek = true;
                  break;
                }
                // 兼容旧格式：直接在 message.content 中
                else if (msg.content && typeof msg.content === 'string' &&
                         mapping[key].id !== undefined) {
                  isDeepSeek = true;
                  break;
                }
              }
            }
            
            if (isDeepSeek) {
              const title = item.title || "unknown";
              const dialogues = extractDialogueFromDeepSeekMapping(mapping);
              if (!dialogues.length) continue;
              const base = sanitizeFilename(title) || "unknown";
              const count = (counter[base] || 0) + 1;
              counter[base] = count;
              const fn = count > 1 ? `${base}_${count}.txt` : `${base}.txt`;
              outputs.push({ filename: fn, content: dialogues.join("\n") });
            } else {
              const title = item.title || "unknown";
              const dialogues = extractDialogueFromMapping(mapping);
              if (!dialogues.length) continue;
              const base = sanitizeFilename(title) || "unknown";
              const count = (counter[base] || 0) + 1;
              counter[base] = count;
              const fn = count > 1 ? `${base}_${count}.txt` : `${base}.txt`;
              outputs.push({ filename: fn, content: dialogues.join("\n") });
            }
          } else {
            // 未知格式
          }
        }
        if (outputs.length === 0) {
          throw new Error("未找到有效的对话内容");
        }
        return outputs;
      }
      
      // ChatGPT对象格式处理
      if (typeof data === "object" && !Array.isArray(data)) {
        for (const title in data) {
          const content = data[title];
          // 添加空值检查
          if (!content || typeof content !== 'object') continue;
          const mapping = content.mapping;
          if (!mapping) continue;
          const dialogues = extractDialogueFromMapping(mapping);
          if (dialogues.length) {
            const fn = sanitizeFilename(title) || "unknown";
            const txt = dialogues.join("\n");
            outputs.push({ filename: fn + ".txt", content: txt });
          }
        }
        if (outputs.length === 0) {
          throw new Error("未找到有效的对话内容");
        }
      } else {
        throw new Error("不支持的 JSON 格式");
      }
      return outputs;
    }

    function showFileContent(file) {
      currentFile.value = file;
      showModal.value = true;
      exactTokens.value = null;
      // 重置选择模式
      isSelectMode.value = false;
      selectedMessages.value = [];
    }

    function closeModal() {
      showModal.value = false;
      currentFile.value = null;
      // 重置选择模式
      isSelectMode.value = false;
      selectedMessages.value = [];
    }

    function handleSearch() {
      if (!searchQuery.value.trim()) {
        filteredOutputs.value = lastOutputs.value;
        searchKeyword.value = ''
        return;
      }
      const query = searchQuery.value.toLowerCase();
      filteredOutputs.value = lastOutputs.value.filter(output => {
        if (output.filename.toLowerCase().includes(query)) return true;
        if (output.content.toLowerCase().includes(query)) return true;
        return false;
      });
      searchKeyword.value = searchQuery.value.trim();
    }

    async function handleFileChange(event) {
      const file = event.target.files[0];
      if (!file) return;
      
      status.value = "正在解析文件...";
      const reader = new FileReader();
      
      reader.onload = function (e) {
        try {
          const data = JSON.parse(e.target.result);
          const outputs = processRawJson(data);
          lastOutputs.value = outputs;
          filteredOutputs.value = outputs;
          // 初始化选择状态
          selectedFiles.value = outputs.map(() => false);
          
          if (outputs.length === 0) {
            ElMessage.error("解析完成，但未找到对话内容");
            return;
          }
          
          status.value = `解析完成，找到 ${outputs.length} 个会话`;
        } catch (err) {
          ElMessage.error(err.message || "解析失败");
          status.value = "解析失败";
        }
      };
      
      reader.readAsText(file, "utf-8");
    }

    async function handleDownload() {
      if (!selectedFiles.value.some(selected => selected)) return;
      
      try {
        status.value = "正在准备导出...";
        
        const selectedOutputs = lastOutputs.value.filter((_, index) => selectedFiles.value[index]);
        
        if (selectedOutputs.length === 1) {
          // 单文件直接下载
          const blob = new Blob([selectedOutputs[0].content], { type: "text/plain" });
          const url = URL.createObjectURL(blob);
          const a = document.createElement("a");
          a.href = url;
          a.download = selectedOutputs[0].filename;
          a.click();
          URL.revokeObjectURL(url);
          status.value = "导出完成";
        } else {
          // 多文件打包为 zip
          status.value = "正在打包文件，请稍候...";
          const zip = new JSZip();
          
          // 添加所有选中的文件到zip
          selectedOutputs.forEach(output => {
            zip.file(output.filename, output.content);
          });
          
          // 生成zip文件
          const content = await zip.generateAsync({type: "blob"});
          const url = URL.createObjectURL(content);
          const a = document.createElement("a");
          a.href = url;
          a.download = "chatgpt_dialogues.zip";
          a.click();
          URL.revokeObjectURL(url);
          
          status.value = `已导出 ${selectedOutputs.length} 个文件到 chatgpt_dialogues.zip`;
        }
      } catch (err) {
        status.value = "导出失败";
      }
    }

    const chatLines = computed(() => {
      if (!currentFile.value?.content) return [];
      
      // 只匹配特定的角色标记
      const rolePattern = /【(model_message|user_message|assistant|user)】/;
      const lines = currentFile.value.content.split('\n');
      const messages = [];
      let currentMessage = '';
      let currentRole = '';

      for (const line of lines) {
        const match = line.match(rolePattern);
        if (match) {
          // 如果已经有消息，保存之前的消息
          if (currentMessage) {
            messages.push(`【${currentRole}】${currentMessage}`);
          }
          // 开始新的消息
          currentRole = match[1];
          currentMessage = line.replace(rolePattern, '').trim();
        } else if (currentRole) {
          // 如果不是角色标记，且当前有角色，则添加到当前消息
          currentMessage += '\n' + line;
        }
      }
      
      // 添加最后一条消息
      if (currentMessage) {
        messages.push(`【${currentRole}】${currentMessage}`);
      }

      return messages.filter(msg => {
        const content = msg.replace(/^【.*?】\s*/, '').trim();
        return content.length > 0;
      });
    });

    function getMessageClass(line) {
      if (line.startsWith('【assistant】') || line.startsWith('【model_message】')) {
        return 'message-left';
      }
      return 'message-right';
    }

    function isLeftMessage(line) {
      return line.startsWith('【assistant】') || line.startsWith('【model_message】');
    }

    function getMessageContent(line, lineIndex) {
      // 提取时间戳
      const timestampMatch = line.match(/\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\]/);
      const timestamp = timestampMatch ? timestampMatch[1] : '';
      
      // 移除角色标记和时间戳
      let content = line.replace(/^【.*?】\s*/, '').trim();
      content = content.replace(/\[\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\]\s*/, '');
      
      // 只在 user_message 类型的消息中移除分割线，且只匹配20个或更多连续的连字符
      const cleanContent = line.startsWith('【user_message】') 
        ? content.replace(/-{20,}/g, '')
        : content;
        
      // 将全角引号转换为半角
      const normalizedContent = cleanContent
        .replace(/['’＇]/g, "'")  // 全角单引号转半角
        .replace(/["＂]/g, '"')  // 全角双引号转半角
        
      let html = marked(normalizedContent, { breaks: true });
      
      if (searchKeyword.value) {
        // 高亮所有关键词
        const reg = new RegExp(searchKeyword.value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&'), 'gi');
        let matchIdx = 0;
        html = html.replace(reg, (m) => {
          const refId = `search-match-${lineIndex}-${matchIdx}`;
          matchIdx++;
          return `<mark data-match-idx="${matchIdx-1}" id="${refId}">${m}</mark>`;
        });
      }
      
      // 添加时间戳到内容前面
      if (timestamp) {
        html = `<span class="timestamp">${timestamp}</span>${html}`;
      }
      
      return html.replace(/\s+$/, '').replace(/\n$/, '');
    }

    const statistics = computed(() => {
      if (!currentFile.value?.content) {
        return {
          totalTokens: 0,
          assistantCount: 0,
          userCount: 0
        };
      }

      const lines = currentFile.value.content.split('\n');
      let totalTokens = 0;
      let assistantCount = 0;
      let userCount = 0;
      let currentContent = '';
      let currentRole = '';

      const countTokens = (text) => {
        // 移除URL和时间戳
        text = text.replace(/https?:\/\/[^\s]+/g, '');
        text = text.replace(/\[\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}\]/g, '');
        
        let count = 0;
        
        // 分割文本为单词、数字和其他字符
        const tokens = text.match(/[a-zA-Z]+|[0-9]+|[\u4e00-\u9fa5]|[^\s]/g) || [];
        
        for (const token of tokens) {
          if (/[a-zA-Z]/.test(token)) {
            // 英文单词，按单词长度计算
            count += Math.ceil(token.length / 4); // 每4个字符算一个token
          } else if (/[0-9]/.test(token)) {
            // 数字，按位数计算
            count += Math.ceil(token.length / 2); // 每2位数字算一个token
          } else if (/[\u4e00-\u9fa5]/.test(token)) {
            // 中文字符
            count += 1;
          } else if (/[\s.,!?;:'"()\-–—…]/.test(token)) {
            // 标点符号和空格
            count += 0.5;
          } else {
            // 其他字符（包括markdown标记）
            count += 0.5;
          }
        }
        
        return Math.round(count);
      };

      for (const line of lines) {
        if (line.startsWith('【assistant】') || line.startsWith('【model_message】')) {
          // 如果之前有内容，先处理之前的内容
          if (currentContent && currentRole) {
            totalTokens += countTokens(currentContent);
            if (currentRole === 'assistant') assistantCount++;
            else userCount++;
          }
          currentRole = 'assistant';
          currentContent = line.replace(/^【.*?】\s*/, '').trim();
        } else if (line.startsWith('【user】') || line.startsWith('【user_message】')) {
          // 如果之前有内容，先处理之前的内容
          if (currentContent && currentRole) {
            totalTokens += countTokens(currentContent);
            if (currentRole === 'assistant') assistantCount++;
            else userCount++;
          }
          currentRole = 'user';
          currentContent = line.replace(/^【.*?】\s*/, '').trim();
        } else if (currentRole) {
          // 如果不是角色标记，且当前有角色，则添加到当前内容
          currentContent += '\n' + line;
        }
      }
      
      // 处理最后一条消息
      if (currentContent && currentRole) {
        totalTokens += countTokens(currentContent);
        if (currentRole === 'assistant') assistantCount++;
        else userCount++;
      }

      return {
        totalTokens,
        assistantCount,
        userCount
      };
    });

    // 预加载编码器
    onMounted(async () => {
      // 设置页面标题
      document.title = '💬 对话记录解析器'

      // 等待页面渲染完成
      await nextTick()
      
      // 延迟一小段时间再开始加载，确保页面完全渲染
      setTimeout(async () => {
        try {
          await preloadEncoder()
        } catch (error) {
          console.error('Failed to preload encoder:', error)
          ElMessage.error('加载失败，请刷新页面重试')
        }
      }, 100)
    })

    async function calculateExactTokens() {
      if (!currentFile.value?.content) return
      
      isCalculating.value = true
      exactTokens.value = null
      
      try {
        // 打印用于计算的内容
        // console.log('用于计算token的内容：', currentFile.value.content)
        const tokens = await calculateTokens(currentFile.value.content)
        if (tokens !== null) {
          exactTokens.value = tokens
        }
      } catch (error) {
        console.error('Token calculation failed:', error)
        ElMessage.error('计算token失败，请稍后重试')
      } finally {
        isCalculating.value = false
      }
    }

    // 切换选择模式
    function toggleSelectMode() {
      isSelectMode.value = !isSelectMode.value;
      if (isSelectMode.value) {
        // 进入选择模式时，初始化选择状态
        selectedMessages.value = chatLines.value.map(() => false);
      } else {
        // 退出选择模式时，清空选择状态
        selectedMessages.value = [];
      }
    }

    // 全选/取消全选消息
    function toggleSelectAllMessages() {
      const shouldSelectAll = !isAllMessagesSelected.value;
      selectedMessages.value = chatLines.value.map(() => shouldSelectAll);
    }

    // 导出选中的消息
    async function exportSelectedMessages() {
      if (!currentFile.value?.filename) {
        ElMessage.error('没有可导出的对话内容');
        return;
      }

      const selectedLines = chatLines.value.filter((_, index) => selectedMessages.value[index]);
      if (selectedLines.length === 0) {
        ElMessage.warning('请选择要导出的消息');
        return;
      }

      try {
        // 创建包含选中消息的内容
        const exportContent = selectedLines.join('\n');
        const blob = new Blob([exportContent], { type: "text/plain;charset=utf-8" });
        const url = URL.createObjectURL(blob);
        const a = document.createElement("a");
        a.href = url;
        
        // 生成文件名
        const originalName = currentFile.value.filename.replace(/\.[^/.]+$/, "");
        a.download = `${originalName}_选中消息.txt`;
        a.click();
        URL.revokeObjectURL(url);
        
        ElMessage.success(`已导出 ${selectedLines.length} 条选中消息`);
      } catch (error) {
        console.error('导出失败:', error);
        ElMessage.error('导出失败，请稍后重试');
      }
    }

    // 新增：处理txt文件上传
    async function handleTxtFileChange(event) {
      const files = event.target.files;
      if (!files || !files.length) return;
      let newOutputs = [];
      for (const file of files) {
        try {
          const text = await file.text();
          // 检查文件内容是否为空或只包含空白字符
          if (!text.trim()) {
            ElMessage.error(`${file.name} 文件内容为空`);
            continue;
          }
          
          // 检查是否包含有效的对话内容
          const lines = text.split('\n');
          const hasValidContent = lines.some(line => {
            const trimmedLine = line.trim();
            return trimmedLine && !trimmedLine.startsWith('-'.repeat(20));
          });
          
          if (!hasValidContent) {
            ElMessage.error(`${file.name} 未包含有效的对话内容`);
            continue;
          }

          // 检查是否包含AI或用户对话
          const hasDialogContent = lines.some(line => {
            const trimmedLine = line.trim();
            return trimmedLine && (
              trimmedLine.startsWith('【assistant】') ||
              trimmedLine.startsWith('【user】') ||
              trimmedLine.startsWith('【model_message】') ||
              trimmedLine.startsWith('【user_message】')
            );
          });

          if (!hasDialogContent) {
            ElMessage.error(`${file.name} 未包含有效的AI或用户对话内容`);
            continue;
          }
          
          // 以文件名为会话名
          newOutputs.push({ filename: file.name, content: text });
        } catch (e) {
          ElMessage.error(`读取${file.name}失败`);
        }
      }
      if (newOutputs.length) {
        // 合并到现有聊天窗口
        lastOutputs.value = [...lastOutputs.value, ...newOutputs];
        filteredOutputs.value = lastOutputs.value;
        selectedFiles.value = lastOutputs.value.map(() => false);
        status.value = `已导入 ${newOutputs.length} 个txt文件`;
      }
      // 清空input，允许重复上传同一文件
      event.target.value = '';
    }

    // 统计所有高亮
    function updateMatchRefs() {
      nextTick(() => {
        matchRefs.value = Array.from(document.querySelectorAll('.modal-body mark[data-match-idx]'));
        matchCount.value = matchRefs.value.length;
        if (matchCount.value > 0) {
          scrollToMatch(currentMatchIndex.value);
        }
      });
    }

    // 滚动到指定高亮
    function scrollToMatch(idx) {
      if (matchRefs.value[idx]) {
        matchRefs.value[idx].scrollIntoView({ behavior: 'smooth', block: 'center' });
        matchRefs.value.forEach((el, i) => {
          el.classList.toggle('active-match', i === idx);
        });
      }
    }

    function gotoPrevMatch() {
      if (matchCount.value <= 1) return;
      currentMatchIndex.value = (currentMatchIndex.value - 1 + matchCount.value) % matchCount.value;
      scrollToMatch(currentMatchIndex.value);
    }
    function gotoNextMatch() {
      if (matchCount.value <= 1) return;
      currentMatchIndex.value = (currentMatchIndex.value + 1) % matchCount.value;
      scrollToMatch(currentMatchIndex.value);
    }

    // 每次弹窗打开或内容变化时，更新高亮和滚动
    watch([showModal, searchKeyword, currentFile], () => {
      if (showModal.value) {
        currentMatchIndex.value = 0;
        updateMatchRefs();
      }
    });

    return {
      outputContent,
      status,
      canDownload,
      lastOutputs,
      selectedFiles,
      isAllSelected,
      showModal,
      currentFile,
      searchQuery,
      filteredOutputs,
      statistics,
      handleFileChange,
      handleDownload,
      showFileContent,
      closeModal,
      toggleSelectAll,
      updateSelection,
      chatLines,
      getMessageClass,
      isLeftMessage,
      getMessageContent,
      handleSearch,
      isCalculating,
      exactTokens,
      calculateExactTokens,
      handleTxtFileChange,
      searchKeyword,
      matchCount,
      currentMatchIndex,
      gotoPrevMatch,
      gotoNextMatch,
      toggleSelectMode,
      toggleSelectAllMessages,
      exportSelectedMessages,
      selectedMessages,
      isSelectMode,
      isAllMessagesSelected,
      hasSelectedMessages,
      selectedMessagesCount
    };
  }
}
</script>

<style scoped>
* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

.container {
  font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
  line-height: 1.6;
  padding: 30px;
  max-width: 800px;
  margin: 0 auto;
  background-color: #fff;
  border-radius: 16px;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
}

h1 {
  color: #8e9aaf;
  margin-bottom: 30px;
  text-align: center;
  font-size: 1.8em;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

h2 {
  font-size: 1.3em;
  margin-bottom: 15px;
  color: #8e9aaf;
  display: flex;
  align-items: center;
  gap: 8px;
}

.file-input-container {
  margin: 20px 0;
  text-align: center;
}

.file-input-label {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  background-color: #e8eaf6;
  color: #7986cb;
  border-radius: 12px;
  cursor: pointer;
  transition: all 0.3s ease;
  font-weight: 500;
}

.file-input-label:hover {
  background-color: #c5cae9;
  transform: translateY(-2px);
}

input[type="file"] {
  display: none;
}

.file-list {
  margin: 20px 0;
  background: #f8f9fa;
  padding: 20px;
  border-radius: 12px;
  border: 1px solid #e8eaf6;
  height: 50vh;
  display: flex;
  flex-direction: column;
}

.search-container {
  margin-bottom: 15px;
}

.search-container :deep(.el-input__wrapper) {
  background-color: white;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  border-radius: 8px;
}

.search-container :deep(.el-input__inner) {
  height: 40px;
}

.search-container :deep(.el-input__prefix) {
  color: #9fa8da;
}

.file-list-content {
  flex-grow: 1;
  overflow-y: auto;
  border: 1px solid #e8eaf6;
  border-radius: 8px;
  background: white;
}

.select-all-container {
  padding: 12px;
  border-bottom: 1px solid #e8eaf6;
  background-color: #f5f6fa;
}

.select-all-label {
  display: flex;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  user-select: none;
  color: #7986cb;
  font-weight: 500;
}

.file-item {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  padding: 12px;
  border-bottom: 1px solid #e8eaf6;
  transition: all 0.2s ease;
}

.file-checkbox {
  display: flex;
  align-items: center;
  cursor: pointer;
  flex-shrink: 0;
}

.file-name {
  flex-grow: 1;
  cursor: pointer;
  display: flex;
  align-items: flex-start;
  gap: 8px;
  color: #5c6bc0;
  word-break: break-word;
  text-align: left;
  min-width: 0;
}

.file-name .el-icon {
  flex-shrink: 0;
  margin-top: 2px;
}

.file-name:hover {
  color: #7986cb;
}

.file-item:hover {
  background-color: #f5f6fa;
}

.empty-state {
  padding: 40px 20px;
  text-align: center;
  color: #9fa8da;
  font-style: italic;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 12px;
}

.empty-state .el-icon {
  font-size: 24px;
}

.button-container {
  text-align: center;
  margin-top: 20px;
}

button {
  display: inline-flex;
  align-items: center;
  gap: 8px;
  padding: 12px 24px;
  background-color: #7986cb;
  color: white;
  border: none;
  border-radius: 12px;
  cursor: pointer;
  font-size: 16px;
  font-weight: 500;
  transition: all 0.3s ease;
}

button:hover {
  background-color: #5c6bc0;
  transform: translateY(-2px);
}

button:disabled {
  background-color: #c5cae9;
  cursor: not-allowed;
  transform: none;
}

.status {
  margin-top: 15px;
  text-align: center;
  color: #9fa8da;
  font-size: 0.9em;
}

.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100vh;
  height: 100dvh; /* 动态视口高度，排除浏览器UI */
  background-color: rgba(0, 0, 0, 0.2);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  backdrop-filter: blur(4px);
}

.modal-content {
  background-color: white;
  border-radius: 16px;
  width: 95%;
  max-width: 800px;
  max-height: 90vh;
  max-height: 90dvh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.1);
  /* 确保内容在安全区域内 */
  margin: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
}

.modal-header {
  padding: 20px;
  border-bottom: 1px solid #e8eaf6;
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.header-top {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 12px;
}

.modal-header h3 {
  margin: 0;
  color: #5c6bc0;
  display: flex;
  align-items: center;
  gap: 8px;
  word-break: break-all;
  font-size: 1.1em;
  flex: 1;
  min-width: 0;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 0;
}

.statistics-note {
  display: flex;
  align-items: center;
  color: #9fa8da;
  cursor: help;
  width: 32px;
  height: 32px;
  justify-content: center;
  border-radius: 4px;
  transition: all 0.2s ease;
}

.statistics-note:hover {
  color: #5c6bc0;
  background-color: #f5f6fa;
}

.statistics-note .el-icon {
  font-size: 16px;
}

.close-button {
  background: none;
  border: none;
  cursor: pointer;
  color: #9fa8da;
  padding: 4px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.2s ease;
  flex-shrink: 0;
  width: 32px;
  height: 32px;
  border-radius: 4px;
}

.close-button:hover {
  color: #5c6bc0;
  background-color: #f5f6fa;
}

.statistics {
  display: flex;
  align-items: center;
  gap: 12px;
  font-size: 0.9em;
  color: #666;
  flex-wrap: wrap;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 4px;
  white-space: nowrap;
  flex-shrink: 0;
}

.stat-label {
  color: #9fa8da;
}

.stat-value {
  color: #5c6bc0;
  font-weight: 500;
}

.token-calc {
  margin-left: auto;
  display: flex;
  align-items: center;
  gap: 8px;
  flex-shrink: 1;
  flex-wrap: wrap;
  min-width: 0;
  justify-content: flex-end;
}

.export-current {
  display: flex;
  align-items: center;
  flex-shrink: 0;
  flex-wrap: wrap;
  gap: 8px;
}

.export-button {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 6px 12px;
  background-color: #e8f5e8;
  color: #4caf50;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.export-button:hover {
  background-color: #c8e6c9;
  transform: translateY(-1px);
}

.export-button:disabled {
  background-color: #f1f8e9;
  color: #81c784;
  cursor: not-allowed;
  transform: none;
}

.mode-toggle-button {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 6px 12px;
  background-color: #e3f2fd;
  color: #1976d2;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.mode-toggle-button:hover {
  background-color: #bbdefb;
  transform: translateY(-1px);
}

.select-all-button {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 6px 12px;
  background-color: #fff3e0;
  color: #f57c00;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.select-all-button:hover {
  background-color: #ffe0b2;
  transform: translateY(-1px);
}

.message-checkbox {
  display: flex;
  align-items: flex-start;
  padding-top: 12px;
  flex-shrink: 0;
}

.message-checkbox input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #7986cb;
  margin: 0;
}

.message-checkbox label {
  cursor: pointer;
  margin: 0;
}

.modal-body {
  padding: 20px;
  overflow-y: auto;
  flex-grow: 1;
  background-color: #f5f6fa;
}

.chat-container {
  display: flex;
  flex-direction: column;
  gap: 16px;
  width: 100%;
  padding: 0 12px;
}

.message-container {
  display: flex;
  align-items: flex-start;
  gap: 12px;
  width: 100%;
  max-width: 100%;
  padding-left: 12px; /* 默认左边距，与选中状态保持一致 */
}

.message-container.selected {
  /* 添加微妙的边框指示，不影响布局 */
  border-left: 3px solid #7986cb;
  padding-left: 9px; /* 补偿边框宽度，保持对齐 */
}

.message-wrapper {
  display: flex;
  gap: 12px;
  flex: 1;
  min-width: 0;
}

.message-left {
  align-self: flex-start;
  padding-right: 20%;
}

.message-right {
  align-self: flex-end;
  flex-direction: row-reverse;
  padding-left: 20%;
}

.avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background-color: #e8eaf6;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}

.message-bubble {
  padding: 12px 16px;
  border-radius: 16px;
  font-size: 0.95em;
  line-height: 1.5;
  word-break: break-word;
  white-space: pre-wrap;
  text-align: left;
  min-width: 60px;
  max-width: 100%;
  display: inline-block;
  position: relative;
}

.message-bubble :deep(.timestamp) {
  font-size: 0.8em;
  opacity: 0.7;
  margin-bottom: 4px;
  display: block;
}

.message-left .message-bubble :deep(.timestamp) {
  color: #7986cb;
}

.message-right .message-bubble :deep(.timestamp) {
  color: #e8eaf6;
}

.message-bubble :deep(p) {
  margin: 0;
  padding: 0;
}

.message-bubble :deep(p:last-child) {
  margin-bottom: 0;
}

.message-bubble :deep(p + p) {
  margin-top: 1em;
}

.message-bubble :deep(em) {
  font-style: italic;
}

.message-bubble :deep(strong) {
  font-weight: bold;
}

.message-bubble :deep(code) {
  background-color: rgba(0, 0, 0, 0.05);
  padding: 2px 4px;
  border-radius: 4px;
  font-family: monospace;
}

.message-bubble :deep(pre) {
  background-color: rgba(0, 0, 0, 0.05);
  padding: 8px;
  border-radius: 4px;
  overflow-x: auto;
  margin: 8px 0;
}

.message-bubble :deep(pre code) {
  background-color: transparent;
  padding: 0;
}

.message-bubble :deep(ul), .message-bubble :deep(ol) {
  margin: 8px 0;
  padding-left: 20px;
}

.message-bubble :deep(blockquote) {
  border-left: 4px solid rgba(0, 0, 0, 0.1);
  margin: 8px 0;
  padding-left: 12px;
  color: rgba(0, 0, 0, 0.6);
}

.message-left .message-bubble {
  background-color: white;
  color: #5c6bc0;
  border: 1px solid #e8eaf6;
  border-top-left-radius: 4px;
}

.message-right .message-bubble {
  background-color: #7986cb;
  color: white;
  border-top-right-radius: 4px;
}

.message-bubble :deep(mark[data-match-idx]) {
  background: #e8eaf6;
  color: #5c6bc0;
  padding: 0 2px;
  border-radius: 2px;
  transition: all 0.2s;
}

.message-bubble :deep(mark[data-match-idx].active-match) {
  background: #c5cae9;
  box-shadow: 0 0 0 2px #7986cb;
}

/* 移动端适配 */
@media screen and (max-width: 768px) {
  .container {
    padding: 20px;
  }

  .modal-content {
    width: 100%;
    max-height: 100vh;
    max-height: 100dvh;
    border-radius: 0;
    margin: 0;
    /* 使用padding来处理安全区域，保持全屏效果 */
    padding-top: max(env(safe-area-inset-top), 0px);
    padding-bottom: max(env(safe-area-inset-bottom), 0px);
  }

  .modal-header {
    padding: 16px;
    gap: 8px;
    /* 在移动端减少顶部padding，因为modal-content已经有安全区域padding */
    padding-top: 12px;
  }

  .header-top {
    gap: 8px;
  }

  .close-button {
    width: 28px;
    height: 28px;
  }

  .statistics {
    font-size: 0.85em;
  }

  .file-input-label {
    padding: 8px 16px;
    font-size: 0.9em;
    gap: 6px;
  }

  .file-input-label + .file-input-label {
    margin-left: 8px;
  }

  .statistics-row {
    gap: 8px;
  }

  .stat-item {
    gap: 2px;
  }

  .token-calc {
    gap: 6px;
  }

  .message-left {
    padding-right: 5%;
  }

  .message-right {
    padding-left: 5%;
  }

  .avatar {
    width: 36px;
    height: 36px;
  }

  .message-bubble {
    font-size: 0.9em;
    padding: 10px 14px;
  }

  .header-actions {
    gap: 4px;
  }

  .statistics-note {
    width: 28px;
    height: 28px;
  }

  .export-current {
    order: 0;
    width: auto;
    justify-content: flex-start;
    margin-top: 0;
    gap: 4px;
    flex: 1;
  }

  .export-button {
    padding: 4px 8px;
    font-size: 0.8em;
    flex: 0 0 auto;
    max-width: none;
    min-width: auto;
    white-space: nowrap;
  }

  .mode-toggle-button,
  .select-all-button {
    padding: 4px 8px;
    font-size: 0.8em;
    flex: 0 0 auto;
    max-width: none;
    min-width: auto;
    white-space: nowrap;
  }

  .token-calc {
    width: auto;
    justify-content: flex-end;
    margin-top: 0;
    margin-left: auto;
    flex: 0 0 auto;
  }

  .message-checkbox input[type="checkbox"] {
    width: 16px;
    height: 16px;
  }

  .message-container.selected {
    /* 移除会导致移位的样式 */
  }

  .message-checkbox {
    padding-top: 10px;
  }
}

@media screen and (max-width: 480px) {
  .container {
    padding: 15px;
  }

  .modal-content {
    max-height: 100vh;
    max-height: 100dvh;
    /* 确保安全区域处理 */
    padding-top: max(env(safe-area-inset-top), 0px);
    padding-bottom: max(env(safe-area-inset-bottom), 0px);
  }

  .modal-header {
    padding: 12px;
  }

  .close-button {
    width: 24px;
    height: 24px;
  }

  .statistics {
    font-size: 0.8em;
  }

  .file-input-label {
    padding: 6px 12px;
    font-size: 0.85em;
    gap: 4px;
  }

  .file-input-label + .file-input-label {
    margin-left: 6px;
  }

  .statistics-row {
    gap: 6px;
  }

  .stat-item {
    font-size: 0.85em;
  }

  .export-current {
    flex-direction: row;
    gap: 4px;
    align-items: center;
    flex: 1;
    justify-content: flex-start;
  }

  .export-button {
    padding: 4px 8px;
    font-size: 0.75em;
    width: auto;
    max-width: none;
    white-space: nowrap;
    flex: 0 0 auto;
  }

  .mode-toggle-button,
  .select-all-button {
    padding: 4px 8px;
    font-size: 0.75em;
    width: auto;
    max-width: none;
    white-space: nowrap;
    flex: 0 0 auto;
  }

  .token-calc {
    width: auto;
    justify-content: flex-end;
    margin-top: 0;
    margin-left: auto;
    flex: 0 0 auto;
    gap: 4px;
  }

  .token-button {
    padding: 4px 8px;
    font-size: 0.75em;
  }

  .message-left {
    padding-right: 2%;
  }

  .message-right {
    padding-left: 2%;
  }

  .avatar {
    width: 32px;
    height: 32px;
  }

  .message-bubble {
    font-size: 0.85em;
    padding: 8px 12px;
  }

  .statistics-note {
    width: 24px;
    height: 24px;
  }

  .message-checkbox input[type="checkbox"] {
    width: 14px;
    height: 14px;
  }

  .message-checkbox {
    padding-top: 8px;
  }
}

/* 优化滚动条样式 */
.modal-body::-webkit-scrollbar {
  width: 6px;
}

.modal-body::-webkit-scrollbar-track {
  background: transparent;
}

.modal-body::-webkit-scrollbar-thumb {
  background: #c5cae9;
  border-radius: 3px;
}

.modal-body::-webkit-scrollbar-thumb:hover {
  background: #9fa8da;
}

.token-button {
  display: inline-flex;
  align-items: center;
  gap: 4px;
  padding: 6px 12px;
  background-color: #e8eaf6;
  color: #5c6bc0;
  border: none;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9em;
  transition: all 0.2s ease;
}

.token-button:hover:not(:disabled) {
  background-color: #c5cae9;
  transform: translateY(-1px);
}

.token-button:disabled {
  background-color: #f5f6fa;
  color: #9fa8da;
  cursor: not-allowed;
  transform: none;
}

.token-value {
  color: #5c6bc0;
  font-weight: 500;
  margin-right: 4px;
}

.token-label {
  color: #9fa8da;
  font-size: 0.9em;
}

.is-loading {
  animation: rotating 2s linear infinite;
}

@keyframes rotating {
  from {
    transform: rotate(0deg);
  }
  to {
    transform: rotate(360deg);
  }
}

@media screen and (max-width: 768px) {
  .token-button {
    padding: 4px 8px;
    font-size: 0.85em;
  }
}

@media screen and (max-width: 480px) {
  .statistics-table {
    font-size: 0.85em;
  }

  .statistics-table td:not(:last-child) {
    padding-right: 6px;
  }

  .token-calc {
    padding-left: 6px;
  }

  .token-button {
    padding: 4px 8px;
    font-size: 0.8em;
  }
}

.search-nav {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  margin-top: 8px;
  margin-bottom: 0;
  width: 100%;
}
.search-nav button {
  background: #e8eaf6;
  color: #5c6bc0;
  border: none;
  border-radius: 6px;
  padding: 4px 12px;
  cursor: pointer;
  font-size: 0.95em;
  transition: all 0.2s;
  min-width: 80px;
}
.search-nav button:disabled {
  background: #f5f6fa;
  color: #b0b0b0;
  cursor: not-allowed;
}
.search-nav span {
  min-width: 60px;
  text-align: center;
  color: #5c6bc0;
}

.statistics-table {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

.statistics-table td {
  padding: 0;
  white-space: nowrap;
  width: auto;
}

.statistics-table td:not(:last-child) {
  padding-right: 12px;
}

.token-calc {
  text-align: right;
  padding-left: 12px;
  width: auto;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 4px;
  white-space: nowrap;
}

@media screen and (max-width: 768px) {
  .statistics-table {
    font-size: 0.9em;
  }

  .statistics-table td:not(:last-child) {
    padding-right: 8px;
  }

  .token-calc {
    padding-left: 8px;
  }

  .token-button {
    padding: 4px 8px;
    font-size: 0.85em;
  }

  .stat-item {
    gap: 2px;
  }
}

@media screen and (max-width: 480px) {
  .statistics-table {
    font-size: 0.85em;
  }

  .statistics-table td:not(:last-child) {
    padding-right: 6px;
  }

  .token-calc {
    padding-left: 6px;
  }

  .token-button {
    padding: 4px 8px;
    font-size: 0.8em;
  }
}
</style> 