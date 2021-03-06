<template>
  <div class="project-list">
    <div class="filter-wrapper">
      <el-input placeholder="请输入内容" v-model="keyword">
        <el-button slot="append" icon="el-icon-search"></el-button>
      </el-input>
    </div>

    <div class="list-wrapper" id="sort">
      <router-link
        :to="`/${linkPathArr[0]}/${item.id}/${linkPathArr[1]}`"
        class="item"
        v-for="item in list"
        v-if="keyword.trim() === '' || item.name.indexOf(keyword) > -1"
        :key="item.id"
      >
        <img v-if="item.picUrl !== ''" :src="item.picUrl | setFileRoot" />
        <img v-else src="@/assets/images/project-bg.png" />
        <div class="name">{{ item.name }}</div>
        <div class="user-info">负责人：{{ item.leader }}</div>
      </router-link>
      <div class="item add-item hover-cursor" v-if="!disableEdit" @click="addProject">
        <i class="el-icon-plus" style="font-size: 24px"></i>
        <div class="action-name">创建新项目</div>
      </div>
    </div>

    <!-- dialog -->
    <el-dialog
      :close-on-click-modal="false"
      title="创建项目"
      :visible.sync="dialogFormVisible"
      @close="resetForm"
      width="540px"
    >
      <el-form :rules="rules" ref="dialogForm" label-position="right" :model="project" :disabled="disableEdit" label-width="120px" style='width: 440px; margin-left:30px;'>
        <el-form-item :label="$t('project.name')" prop="name">
          <el-input v-model="project.name"></el-input>
        </el-form-item>
        <el-form-item :label="$t('project.leader')" prop="leader">
          <el-input v-model="project.leader"></el-input>
        </el-form-item>
        <el-form-item :label="$t('project.place')" prop="place">
          <el-input v-model="project.place"></el-input>
        </el-form-item>
        <el-form-item label="产值(万元)：" prop="price">
          <el-input-number v-model="project.price" :min="1" label="请输入产值"></el-input-number>
        </el-form-item>
        <el-form-item :label="$t('project.startDate')" prop="startDate">
          <el-date-picker
            v-model="project.dateValue"
            type="daterange"
            align="right"
            unlink-panels
            range-separator="至"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
            class="filter-item"
            value-format="yyyy-MM-dd HH:mm:ss"
            @change="handleDateChange"
          >
          </el-date-picker>
        </el-form-item>
        <el-form-item :label="$t('project.description')" prop="description">
          <el-input type="textarea" v-model="project.description"></el-input>
        </el-form-item>
        <el-form-item :label="$t('project.picFile')" prop="picFile">
          <div class="upload-file-wrapper flex-column">
            <el-button type="primary" style="width: 100px">
              <input type="file" ref="picInput" accept="image/*" @change="handlePicChange" />
              <span>上传</span>
            </el-button>
            <div style="font-size: 14px;">只能上传jpg/png文件，且不超过500kb</div>
            <img v-if="uploadFileSrc" class="file" :src="uploadFileSrc" />
          </div>
        </el-form-item>
        <el-form-item label="模型类型">
          <el-select v-model="model.modelType">
            <el-option :value="0" label="单体模型"></el-option>
            <el-option :value="1" label="集成模型"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="模型上传">
          <div class="upload-file-wrapper flex-column">
            <div class="flex-row">
              <el-button type="primary" style="margin-right: 15px;">
                <input type="file" ref="picInput" @change="handleModelChange" />
                <span>点击上传</span>
              </el-button>
            </div>
            <span>{{model.fileName !== null ? model.fileName : ''}}</span>
            <div style="font-size: 12px;">只能上传一个模型文件。</div>
          </div>
        </el-form-item>
        <el-form-item label="模型上传进度" v-if="progress > 0">
          <el-progress :text-inside="true" :stroke-width="18" :percentage="progress" :status="progress === 100 ? 'success' : 'text'"></el-progress>
        </el-form-item>
      </el-form>
      <div slot="footer">
        <el-button @click="closeDialog">{{$t('btn.cancel')}}</el-button>
        <el-button type="primary" :disabled="disableEdit" @click="handleSave">{{$t('btn.comfirm')}}</el-button>
      </div>
    </el-dialog>
    <!-- /dialog -->
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import { addUserIndexList } from '@/api/sort';
import { getProjectList, addProject, uploadModelFile } from '@/api/projectManage';

export default {
  name: 'ProjectList',
  data() {
    return {
      keyword: '',
      list: [],
      project: {
        name: '',
        leader: '',
        place: '',
        startDate: null,
        finishedDate: null,
        picFile: '',
        price: 0,
      },
      // date
      dateValue: '',
      // dialog
      dialogFormVisible: false,
      // rules
      rules: {
        name: [{ required: true, message: `${this.$t('project.name')}${this.$t('message.notEmpty')}`, trigger: 'blur' }],
        leader: [{ required: true, message: `${this.$t('project.leader')}${this.$t('message.notEmpty')}`, trigger: 'blur' }],
        place: [{ required: true, message: `${this.$t('project.place')}${this.$t('message.notEmpty')}`, trigger: 'blur' }],
        startDate: [{ required: true, message: `${this.$t('project.startDate')}${this.$t('message.notEmpty')}`, trigger: 'blur' }],
        description: [{ required: true, message: `${this.$t('project.description')}${this.$t('message.notEmpty')}`, trigger: 'blur' }],
        picFile: [{ required: true, message: `${this.$t('project.picFile')}${this.$t('message.notEmpty')}`, trigger: 'change' }],
        price: [{ required: true, message: `产值${this.$t('message.notEmpty')}`, trigger: 'blur' }],
      },
      uploadFileSrc: null,
      loading: false,
      // 模型
      model: {
        modelType: 0,
        file: null,
        fileName: '',
      },
      // 进度条
      progress: 0,
    };
  },
  computed: {
    linkPathArr() {
      const routers = this.$store.getters.permission_routers;
      let linkPathArr = ['project', 'project_detail_info'];

      for (let i = 0; i < routers.length; i += 1) {
        const router = routers[i];
        if (!router.hidden && router.children && router.children.length > 0) {
          linkPathArr = [router.name, router.children[0].name];
          break;
        }
      }

      return linkPathArr;
    },
    ...mapGetters({
      userMsg: 'userInfo',
    }),
  },
  created() {
    this.creatList();
  },
  methods: {
    creatList() {
      getProjectList({ pageIndex: -1 }).then((res) => {
        const { data } = res;
        this.list = data;
        this.handleMove();
      });
    },
    getList() {
      getProjectList({ pageIndex: -1 }).then((res) => {
        const { data } = res;
        this.list = data;
      });
    },
    addProject() {
      this.dialogFormVisible = true;
    },
    handleSave() {
      this.$refs.dialogForm.validate((valid) => {
        if (valid) {
          addProject(this.project).then((projectData) => {
            if (this.model.file !== null) {
              const myConfig = {
                onUploadProgress: (progressEvent) => {
                  if (progressEvent.lengthComputable) {
                    this.progress = Number(((progressEvent.loaded / progressEvent.total) * 100).toFixed(2));
                  }
                },
              };
              uploadModelFile({
                modelFile: this.model.file,
                projectId: projectData.data.data.id,
                modeType: this.model.modelType,
              }, myConfig).then((data) => {
                if (data && this.progress === 100) {
                  this.dialogFormVisible = false;
                  this.getList();
                }
              });
            } else {
              this.dialogFormVisible = false;
              this.getList();
            }
          });
        }
      });
    },
    closeDialog() {
      this.model = {
        modelType: 0,
        file: null,
        fileName: '',
      };
      this.dialogFormVisible = false;
    },
    handleModelChange(e) {
      const file = e.target.files[0];
      this.model.file = file;
      this.model.fileName = file.name;
    },
    handlePicChange(e) {
      const files = e.target.files;
      const file = files[0];
      const self = this;

      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = function () {
        self.uploadFileSrc = this.result;
        self.project.picFile = file;
      };
    },
    handleDateChange(dateArr) {
      if (dateArr) {
        this.project.startDate = dateArr[0];
        this.project.finishedDate = dateArr[1];
      } else {
        this.project.startDate = null;
        this.project.finishedDate = null;
      }
    },
    resetForm() {
      // reset file src
      this.uploadFileSrc = null;
      this.$refs.picInput.value = null;
      this.$refs.dialogForm.resetFields();
      this.$refs.dialogForm.$el.reset();
    },
    handleMove() {
      const msg = this;
      const tbody = document.getElementById('sort');
      msg.$Sortable.create(tbody, {
        animation: 180,
        delay: 0,
        handle: '.item',
        onEnd({ newIndex, oldIndex }) {
          if (newIndex !== oldIndex) {
            const tableData = JSON.parse(JSON.stringify(msg.list));
            // 获取被删除的数组
            const currRow = tableData.splice(oldIndex, 1)[0];
            // 插入该数组
            tableData.splice(newIndex, 0, currRow);
            if (tableData.length > 0) {
              const sortResult = [];
              for (const key in tableData) {
                if (key && tableData[key] !== undefined) {
                  const tableDataId = `${tableData[key].id}`;
                  const temp = {
                    userId: msg.userMsg.id,
                    aboutId: tableDataId,
                    aboutType: '6',
                    indexs: (Number(key) + 1),
                  };
                  sortResult.push(temp);
                }
              }
              addUserIndexList({ newList: JSON.stringify(sortResult) }).then(() => {
                msg.getList();
                msg.$message({
                  type: 'success',
                  message: msg.$t('message.operationSuccess'),
                });
              });
            }
          }
        },
      });
    },
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>
  .filter-wrapper {
    margin: 24px 0;
    width: 490px;
  }

  .list-wrapper {
    display: flex;
    flex-wrap: wrap;

    .item {
      position: relative;
      margin-right: 20px;
      margin-bottom: 20px;
      width: 280px;
      height: 180px;
      color: #ffffff;
      background-color: #EBEEF5;

      img {
        width: 100%;
        height: 100%;
        object-fit: cover;
      }

      .name {
        position: absolute;
        left: 28px;
        bottom: 42px;
      }

      .user-info {
        position: absolute;
        left: 28px;
        bottom: 20px;
        font-size: 14px;
      }
    }

    .add-item {
      display: flex;
      justify-content: center;
      flex-direction: column;
      align-items: center;
      color: #409EFF;

      .action-name {
        margin-top: 15px;
      }
    }
  }

  // media
  @media (max-width: 768px) {
    .filter-wrapper {
      width: 100%;
    }

    .list-wrapper {
      .item {
        margin-right: 0;
        width: 100%;
      }
    }
  }
</style>
