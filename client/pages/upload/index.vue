<template>
  <div id="page-wrapper" class="page p-1 sm:p-6 overflow-y-auto" :class="streamLibraryItem ? 'streaming' : ''">
    <div class="w-full max-w-6xl mx-auto">
      <!-- Library & folder picker -->
      <div class="flex flex-wrap my-6 md:-mx-2">
        <div class="w-full md:w-1/5 px-2">
          <!-- <ui-dropdown v-model="selectedLibraryId" :items="libraryItems" :label="$strings.LabelLibrary" :disabled="!!items.length" @input="libraryChanged" /> -->
          <ui-dropdown v-model="selectedLibraryId" :items="libraryItems" :label="$strings.LabelLibrary" :disabled="readyToUpload" @input="libraryChanged" />
        </div>
        <div class="w-full md:w-3/5 px-2">
          <!-- <ui-dropdown v-model="selectedFolderId" :items="folderItems" :disabled="!selectedLibraryId || !!items.length" :label="$strings.LabelFolder" /> -->
          <ui-dropdown v-model="selectedFolderId" :items="folderItems" :disabled="!selectedLibraryId || readyToUpload" :label="$strings.LabelFolder" />
        </div>
        <div class="w-full md:w-1/5 px-2">
          <ui-text-input-with-label :value="selectedLibraryMediaType" readonly :label="$strings.LabelMediaType" />
        </div>
      </div>

      <div v-if="!selectedLibraryIsPodcast" class="flex items-center mb-6 px-2 md:px-0">
        <label class="flex cursor-pointer pt-4">
          <ui-toggle-switch v-model="fetchMetadata.enabled" class="inline-flex" />
          <span class="pl-2 text-base">{{ $strings.LabelAutoFetchMetadata }}</span>
        </label>
        <ui-tooltip :text="$strings.LabelAutoFetchMetadataHelp" class="inline-flex pt-4">
          <span class="pl-1 material-symbols icon-text text-sm cursor-pointer">info</span>
        </ui-tooltip>

        <div class="grow ml-4">
          <ui-dropdown v-model="fetchMetadata.provider" :items="providers" :label="$strings.LabelProvider" />
        </div>
      </div>

      <widgets-alert v-if="error" type="error">
        <p class="text-lg">{{ error }}</p>
      </widgets-alert>

      <!-- Picker display 这里进行修改  只有当vtt上传之后才显示为true 才会切换到上传区域-->   
      <div v-if="!readyToUpload" class="w-full mx-auto border border-white/20 px-4 md:px-12 pt-12 pb-4 my-12 relative" :class="isDragging ? 'bg-primary/40' : 'border-dashed'">
        <p class="text-2xl text-center">{{ isDragging ? $strings.LabelUploaderDropFiles : isIOS ? $strings.LabelUploaderDragAndDropFilesOnly : $strings.LabelUploaderDragAndDrop }}</p>
        <p class="text-center text-sm my-5">{{ $strings.MessageOr }}</p>
        <div class="w-full max-w-xl mx-auto">
          <div class="flex">
            <ui-btn class="w-full mx-1" @click="openFilePicker">{{ $strings.ButtonChooseFiles }}</ui-btn>
            <ui-btn v-if="!isIOS" class="w-full mx-1" @click="openFolderPicker">{{ $strings.ButtonChooseAFolder }} </ui-btn>
            <!-- new button for vtt -->
            <ui-btn class="w-full mx-1" @click="openVttFilePicker">Choose VTT File</ui-btn>
          </div>
        </div>
        <div class="pt-8 text-center">
          <p class="text-xs text-white/50 font-mono mb-4">
            <strong>{{ $strings.LabelSupportedFileTypes }}: </strong>{{ inputAccept.join(', ') }}
          </p>

          <p class="text-sm text-white/70">
            <span v-if="!isIOS">{{ $strings.NoteUploaderFoldersWithMediaFiles }}</span> <span v-if="selectedLibraryMediaType === 'book'">{{ $strings.NoteUploaderOnlyAudioFiles }}</span>
          </p>
        </div>
      </div>
      <!-- Item list header -->
      <div v-else class="w-full flex items-center pb-4 border-b border-white/10">
        <p class="text-lg lowercase">{{ items.length === 1 ? `1 ${$strings.LabelItem}` : $getString('LabelXItems', [items.length]) }}</p>
        <div class="grow" />
        <ui-btn :disabled="processing" small @click="reset">{{ $strings.ButtonReset }}</ui-btn>
      </div>

      <!-- Alerts -->
      <widgets-alert v-if="!items.length && !uploadReady" type="error" class="my-4">
        <p class="text-lg">{{ $strings.MessageNoItemsFound }}</p>
      </widgets-alert>
      <widgets-alert v-if="ignoredFiles.length" type="warning" class="my-4">
        <div class="w-full pr-12">
          <p class="text-base mb-1">{{ $strings.NoteUploaderUnsupportedFiles }}</p>
          <tables-uploaded-files-table :files="ignoredFiles" :title="$strings.HeaderIgnoredFiles" class="text-white" />
          <p class="text-xs text-white/50 font-mono pt-1">
            <strong>{{ $strings.LabelSupportedFileTypes }}: </strong>{{ inputAccept.join(', ') }}
          </p>
        </div>
      </widgets-alert>

      <!-- Item Upload cards -->
      <cards-item-upload-card v-for="item in items" v-if = "readyToUpload" :key="item.index" :ref="`itemCard-${item.index}`" :media-type="selectedLibraryMediaType" :item="item" :provider="fetchMetadata.provider" :processing="processing" @remove="removeItem(item)" />

      <!-- Upload/Reset btns -->
      <div v-show="readyToUpload" class="flex justify-end pb-8 pt-4">
        <ui-btn v-if="!uploadFinished" color="bg-success" :loading="processing" @click="submit">{{ $strings.ButtonUpload }}</ui-btn>
        <ui-btn v-else @click="reset">{{ $strings.ButtonReset }}</ui-btn>
      </div>
    </div>

     <input ref="fileInput" type="file" multiple :accept="isIOS ? '' : inputAccept.join(',')" class="hidden" @change="inputChanged" />
    <input ref="fileFolderInput" type="file" webkitdirectory multiple :accept="inputAccept.join(',')" class="hidden" @change="inputChanged" v-if="!isIOS" />
    <!-- 隐藏的 VTT 文件选择框 -->
    <input ref="vttFileInput" type="file" accept=".vtt" @change="inputChangedVtt" style="display:none">

  </div>
</template>

<script>
import Path from 'path'
import uploadHelpers from '@/mixins/uploadHelpers'

export default {
  mixins: [uploadHelpers],
  data() {
    return {
      isDragging: false,
      error: '',
      items: [],
      ignoredFiles: [],
      selectedLibraryId: null,
      selectedFolderId: null,
      processing: false,
      uploadFinished: false,
      readyToUpload: false,
      fetchMetadata: {
        enabled: false,
        provider: null
      },
    metadataPending: false,
    pendingVttFiles: []

    }
  },
  watch: {
    selectedLibrary(newVal) {
      if (newVal && !this.selectedFolderId) {
        this.setDefaultFolder()
        this.setMetadataProvider()
      }
    },

  readyToUpload(val) {
    console.log('[watcher] readyToUpload ->', val, 'items.length=', this.items && this.items.length);
    console.log('[watcher] items snapshot:', JSON.parse(JSON.stringify(this.items || [])));
  }
  },
  computed: {
    inputAccept() {
      var extensions = []
      Object.values(this.$constants.SupportedFileTypes).forEach((types) => {
        extensions = extensions.concat(types.map((t) => `.${t}`))
      })
      return extensions
    },
    isIOS() {
      const ua = window.navigator.userAgent
      return /iPad|iPhone|iPod/.test(ua) && !window.MSStream
    },
    streamLibraryItem() {
      return this.$store.state.streamLibraryItem
    },
    libraries() {
      return this.$store.state.libraries.libraries
    },
    libraryItems() {
      return this.libraries.map((lib) => {
        return {
          value: lib.id,
          text: lib.name
        }
      })
    },
    selectedLibrary() {

      return this.libraries.find((lib) => lib.id === this.selectedLibraryId)
    },
    selectedLibraryMediaType() {
      //debugger
      return this.selectedLibrary ? this.selectedLibrary.mediaType : null
    },
    selectedLibraryIsPodcast() {
      return this.selectedLibraryMediaType === 'podcast'
    },
    providers() {
      if (this.selectedLibraryIsPodcast) return this.$store.state.scanners.podcastProviders
      return this.$store.state.scanners.providers
    },
    canFetchMetadata() {
      return !this.selectedLibraryIsPodcast && this.fetchMetadata.enabled
    },
    selectedFolder() {
      if (!this.selectedLibrary) return null
      return this.selectedLibrary.folders.find((fold) => fold.id === this.selectedFolderId)
    },
    folderItems() {
      if (!this.selectedLibrary) return []
      return this.selectedLibrary.folders.map((fold) => {
        return {
          value: fold.id,
          text: fold.fullPath
        }
      })
    },
    uploadReady() {
      return !this.items.length && !this.ignoredFiles.length && !this.uploadFinished
    }
  },
  methods: {

    // 把 pending VTT 附加到已存在的 audio item（按文件名或第一个 audio）
attachPendingVttFiles() {
    if (!this.pendingVttFiles || !this.pendingVttFiles.length) return;
    console.log('[attachPendingVttFiles] attaching', this.pendingVttFiles.map(f=>f.name));
    for (const vttFile of this.pendingVttFiles.slice()) {
      // 尝试按基名匹配音频
      const base = vttFile.name.replace(/\.vtt$/i, '');
      let audioItemIndex = this.items.findIndex(it => {
        const titleBase = (it.title || '').replace(/\.[^/.]+$/, '');
        return titleBase === base;
      });
      // 若无精确匹配且只有一个 audio，则附到第一个 audio
      if (audioItemIndex < 0) {
        const singleAudioIndex = this.items.findIndex(it => it.type === 'audio' || (it.files && it.files.some(f => /\.(mp3|m4a|wav|flac|aac)$/i.test(f.name))));
        if (singleAudioIndex >= 0 && this.items.filter(it => it.type === 'audio' || (it.files && it.files.some(f => /\.(mp3|m4a|wav|flac|aac)$/i.test(f.name)))).length === 1) {
          audioItemIndex = singleAudioIndex;
        }
      }
      if (audioItemIndex >= 0) {
        const audioItem = this.items[audioItemIndex];
        const audioBaseName = (audioItem.title || '').replace(/\.[^/.]+$/, '');
        const newVttName = audioBaseName ? (audioBaseName + '.vtt') : vttFile.name;
        const renamedVttFile = new File([vttFile], newVttName, { type: vttFile.type });
        const newFiles = (audioItem.files || []).concat([renamedVttFile]);
        const newItem = Object.assign({}, audioItem, { files: newFiles });
        if (this.$set) {
          this.$set(this.items, audioItemIndex, newItem);
        } else {
          this.items[audioItemIndex].files = newFiles;
        }
        console.log('[attachPendingVttFiles] attached', newVttName, 'to item index', audioItemIndex);
        // 从 pending 列表移除已处理项
        const idx = this.pendingVttFiles.indexOf(vttFile);
        if (idx >= 0) this.pendingVttFiles.splice(idx, 1);
      } else {
        console.warn('[attachPendingVttFiles] no matching audio for', vttFile.name);
      }
    }
    // 处理完之后检查是否可以切换为 ready
    this.$nextTick(() => {
      this.checkReadyToUpload();
    });
  },

    async attemptMetadataFetch() {
     if (!this.canFetchMetadata) {
       return
     }
     console.log('attemptMetadataFetch() called - placeholder implementation')
     // 在此处实现真实的请求/合并逻辑；当前为占位，立即返回
     return
   },

  checkReadyToUpload() {
    // 只有当每个音频 item 都有对应 .vtt 文件时才允许上传
    const audioItems = this.items.filter(it => it.type === 'audio' || (it.files && it.files.some(f => /\.(mp3|m4a|wav|flac|aac)$/i.test(f.name))));
    if (!audioItems.length) {
      this.readyToUpload = false;
      console.log('[checkReadyToUpload] no audio items -> readyToUpload=false');
      return;
    }

    // 对每个 audio item 检查是否存在 .vtt 文件（可以放在同一 item）
    const allHaveVtt = audioItems.every(it => {
      if (!it.files || !it.files.length) return false;
      return it.files.some(f => f.name.toLowerCase().endsWith('.vtt'));
    });

    this.readyToUpload = !!allHaveVtt;
    console.log('[checkReadyToUpload] audioItems:', audioItems.length, 'allHaveVtt:', allHaveVtt, '=> readyToUpload=', this.readyToUpload);
  },

    libraryChanged() {
      if (!this.selectedLibrary && this.selectedFolderId) {
        this.selectedFolderId = null
      } else if (this.selectedFolderId) {
        if (!this.selectedLibrary.folders.find((fold) => fold.id === this.selectedFolderId)) {
          this.selectedFolderId = null
        }
      }
      this.setDefaultFolder()
      this.setMetadataProvider()
    },
    setDefaultFolder() {
      if (!this.selectedFolderId && this.selectedLibrary && this.selectedLibrary.folders.length) {
        this.selectedFolderId = this.selectedLibrary.folders[0].id
      }
    },
    setMetadataProvider() {
      this.fetchMetadata.provider ||= this.$store.getters['libraries/getLibraryProvider'](this.selectedLibraryId)
    },
    removeItem(item) {
      this.items = this.items.filter((b) => b.index !== item.index)
      if (!this.items.length) {
        this.reset()
      }
    },
    reset() {
      this.error = ''
      this.items = []
      this.ignoredFiles = []
      this.uploadFinished = false
      if (this.$refs.fileInput) this.$refs.fileInput.value = ''
      if (this.$refs.fileFolderInput) this.$refs.fileFolderInput.value = ''
    },
    openFilePicker() {
      console.log('[openFilePicker] invoked, $refs.fileInput=', !!this.$refs.fileInput)
      if (this.$refs.fileInput) this.$refs.fileInput.click()
    },
    openFolderPicker() {
      if (this.$refs.fileFolderInput) this.$refs.fileFolderInput.click()
    },

    openVttFilePicker() {
      console.log('[openVttFilePicker] invoked, $refs.vttFileInput=', !!this.$refs.vttFileInput)
      if (this.$refs.vttFileInput) {
        this.$refs.vttFileInput.click();
      }
    },

  inputChangedVtt(e) {
    console.log('[inputChangedVtt] files:', e?.target?.files && e.target.files.length);
    if (!e.target || !e.target.files || !e.target.files.length) return;
    const vttFile = e.target.files[0];
    console.log('[inputChangedVtt] selected vtt:', vttFile.name);

    // 确保 items 存在且有音频文件
    if (!this.items.length) {
        console.warn('[inputChangedVtt] No items found to attach VTT to');
        this.$toast.error('请先选择音频文件');
        return;
    }

    const audioItem = this.items[0];
    console.log('[inputChangedVtt] Current audio item:', {
        type: audioItem.type,
        files: audioItem.files?.map(f => f.name)
    });

    // 确保 files 数组存在
    if (!audioItem.files) {
        audioItem.files = [];
    }

    // 创建新的 VTT 文件（保持原始名称）
    const newVttFile = new File([vttFile], vttFile.name, { type: 'text/vtt' });

    // 直接修改 files 数组
    audioItem.files.push(newVttFile);

    // 通过 $set 触发响应式更新
    this.$set(this.items, 0, {
        ...audioItem,
        files: [...audioItem.files]
    });

    console.log('[inputChangedVtt] After adding VTT:', {
        filesCount: this.items[0].files.length,
        fileNames: this.items[0].files.map(f => f.name)
    });

    // 确保更新后检查状态
    this.$nextTick(() => {
        this.checkReadyToUpload();
        console.log('[inputChangedVtt] After update readyToUpload=', this.readyToUpload);
    });
},



    isDraggingFile(e) {
      // Checks dragging file or folder and not an element on the page
      var dt = e.dataTransfer || {}
      return dt.types && dt.types.indexOf('Files') >= 0
    },
    dragenter(e) {
      e.preventDefault()
      if (this.uploadReady && this.isDraggingFile(e) && !this.isDragging) {
        this.isDragging = true
      }
    },
    dragleave(e) {
      e.preventDefault()
      if (!e.fromElement && this.isDragging) {
        this.isDragging = false
      }
    },
    dragover(e) {
      // This is required to catch the drop event
      e.preventDefault()
    },
    async drop(e) {
      e.preventDefault()
      this.isDragging = false
      var items = e.dataTransfer.items || []

      var itemResults = await this.uploadHelpers.getItemsFromDrop(items, this.selectedLibraryMediaType)
      this.onItemsSelected(itemResults)
    },

  // make inputChanged async and await getItemsFromPicker/onItemsSelected
async inputChanged(e) {
    console.log('[inputChanged] files:', e?.target?.files && e.target.files.length);
    if (!e.target || !e.target.files) return;
    var _files = Array.from(e.target.files);
    console.log('[inputChanged] incoming file names:', _files.map(f => f.name));
    
    // 只保留真正的媒体文件（示例：排除 .vtt）
    _files = _files.filter(f => !f.name.toLowerCase().endsWith('.vtt'));
    console.log('[inputChanged] filtered file names:', _files.map(f => f.name));
    if (!_files || !_files.length) return;

    try {
        // uploadHelpers.getItemsFromPicker 可能返回 Promise -> await 它
        let itemResults = this.uploadHelpers.getItemsFromPicker(_files, this.selectedLibraryMediaType);
        if (itemResults && typeof itemResults.then === 'function') {
            console.log('[inputChanged] getItemsFromPicker returned Promise, awaiting');
            itemResults = await itemResults;
        }
        console.log('[inputChanged] itemResults:', itemResults);

        // 确保等待 onItemsSelected 完成
        if (this.onItemsSelected) {
            const res = await this.onItemsSelected(itemResults);
            console.log('[inputChanged] onItemsSelected completed, items length:', this.items.length);
        }

        // 使用 nextTick 确保 Vue 更新完成后再检查状态
        await this.$nextTick();
        this.checkReadyToUpload();
        console.log('[inputChanged] after checkReadyToUpload, readyToUpload=', this.readyToUpload);
    } catch (err) {
        console.error('[inputChanged] error:', err);
        this.error = err.message || 'Failed to process files';
    }
},


  // make onItemsSelected async-safe
  async onItemsSelected(itemResults) {
    if (!itemResults) return false;
    console.log("selected audio is ",itemResults);
    const ok = this.itemSelectionSuccessful(itemResults);
    if (ok) {
      // attemptMetadataFetch 可能 async
      if (this.attemptMetadataFetch) {
        const r = this.attemptMetadataFetch();
        if (r && typeof r.then === 'function') await r;
      }
      // attachPendingVttFiles and readiness check are called inside itemSelectionSuccessful, but ensure final check
      this.checkReadyToUpload();
      return true;
    }
    return false;
  },

itemSelectionSuccessful(itemResults) {
    console.log('[itemSelectionSuccessful] raw itemResults:', itemResults);

    if (itemResults && itemResults.error) {
      this.error = itemResults.error;
      this.items = [];
      this.ignoredFiles = [];
      return false;
    }

    // 确保 items 正确初始化
    if (itemResults.items && itemResults.items.length) {
      this.items = itemResults.items.map(item => ({
        ...item,
        type: 'audio',  // 明确设置类型
        files: item.itemFiles || item.files || [],  // 确保有 files 数组
      }));
    } else {
      this.items = [];
    }
    
    console.log('[itemSelectionSuccessful] processed items:', this.items);
    this.ignoredFiles = itemResults.ignoredFiles || [];

    // 如果之前有 pending VTT，则尝试附加
    try { this.attachPendingVttFiles(); } catch (err) { console.error('attachPendingVttFiles failed', err); }

    // call readiness check after items are assigned
    try { this.checkReadyToUpload(); } catch (err) { console.error('checkReadyToUpload failed', err); }
    // 5. 最终状态快照
    console.log('[itemSelectionSuccessful] final items structure:', JSON.stringify(this.items.map(item => ({
        type: item.type,
        title: item.title,
        filesCount: item.files.length,
        fileNames: item.files.map(f => f.name || f.path || '(no-name)')
    })), null, 2));
    return true;
},

    updateItemCardStatus(index, status) {
      var ref = this.$refs[`itemCard-${index}`]
      if (ref && ref.length) ref = ref[0]
      if (!ref) {
        console.error('Book card ref not found', index, this.$refs)
      } else {
        ref.setUploadStatus(status)
      }
    },
    async uploadItem(item) {
      var form = new FormData()
      form.set('title', item.title)
      if (!this.selectedLibraryIsPodcast) {
        form.set('author', item.author || '')
        form.set('series', item.series || '')
      }
      form.set('library', this.selectedLibraryId)
      form.set('folder', this.selectedFolderId)

      // var index = 0
      // item.files.forEach((file) => {
      //   form.set(`${index++}`, file)
      // })

      // 文件部分
      item.files.forEach((file, idx) => {
        // 用 append 保留多个同名字段
        form.append(`file${idx}`, file, file.name);
      });

      return this.$axios
        .$post('/api/upload', form)
        .then(() => true)
        .catch((error) => {
          console.error('Failed to upload item', error)
          this.$toast.error(error.response?.data || 'Oops, something went wrong...')
          return false
        })
    },
    validateItems() {
      var itemData = []
      for (var item of this.items) {
        var itemref = this.$refs[`itemCard-${item.index}`]
        if (itemref && itemref.length) itemref = itemref[0]

        if (!itemref) {
          console.error('Invalid item index no ref', item.index, this.$refs.itemCard)
          return false
        } else {
          var data = itemref.getData()
          if (!data) {
            return false
          }
          itemData.push(data)
        }
      }
      return itemData
    },
    async submit() {
      //console.trace('submit() called');
      if (!this.selectedFolderId || !this.selectedLibraryId) {
        this.$toast.error('Must select library and folder')
        document.getElementById('page-wrapper').scroll({ top: 0, left: 0, behavior: 'smooth' })
        return
      }

      if (this.metadataPending) {
        this.attemptMetadataFetch();
        this.metadataPending = false;
      }

      const items = this.validateItems()
      if (!items) {
        this.$toast.error('Some invalid items')
        return
      }
      this.processing = true

      const itemsToUpload = []

      // Check if path already exists before starting upload
      //  uploading fails if path already exists
      for (const item of items) {
        const exists = await this.$axios
          .$post(`/api/filesystem/pathexists`, { directory: item.directory, folderPath: this.selectedFolder.fullPath })
          .then((data) => {
            if (data.exists) {
              if (data.libraryItemTitle) {
                this.$toast.error(this.$getString('ToastUploaderItemExistsInSubdirectoryError', [data.libraryItemTitle]))
              } else {
                this.$toast.error(this.$getString('ToastUploaderFilepathExistsError', [Path.join(this.selectedFolder.fullPath, item.directory)]))
              }
            }
            return data.exists
          })
          .catch((error) => {
            console.error('Failed to check if filepath exists', error)
            return false
          })
        if (!exists) {
          itemsToUpload.push(item)
        }
      }

      for (const item of itemsToUpload) {
        this.updateItemCardStatus(item.index, 'uploading')
        const result = await this.uploadItem(item)
        this.updateItemCardStatus(item.index, result ? 'success' : 'failed')
      }
      this.processing = false
      this.uploadFinished = true
    }
  },
  mounted() {
    this.selectedLibraryId = this.$store.state.libraries.currentLibraryId
    this.setMetadataProvider()

    this.setDefaultFolder()
    window.addEventListener('dragenter', this.dragenter)
    window.addEventListener('dragleave', this.dragleave)
    window.addEventListener('dragover', this.dragover)
    window.addEventListener('drop', this.drop)
  },
  beforeDestroy() {
    window.removeEventListener('dragenter', this.dragenter)
    window.removeEventListener('dragleave', this.dragleave)
    window.removeEventListener('dragover', this.dragover)
    window.removeEventListener('drop', this.drop)
  }
}
</script>
