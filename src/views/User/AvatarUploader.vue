<template>
  <!-- <img style="position:absolute; z-index:1;left:40px;"
          width="50px" src="/img/camera.png"> -->
  <div class="example-avatar">
    <div v-show="$refs.upload && $refs.upload.dropActive" class="drop-active">
      <h3>拖动图片到这里上传</h3>
    </div>
    <div class="avatar-upload"  v-show="!edit">
      <div class="text-center p-2">
        <label for="avatar">
          <img :src="files.length ? files[0].url : 'https://www.gravatar.com/avatar/default?s=200&r=pg&d=mm'"  class="rounded-circle" />
          <h4 class="pt-2">或者<br/>拖动图片到这里上传</h4>
        </label>
      </div>
      <div class="text-center p-2">
        <file-upload
          extensions="gif,jpg,jpeg,png,webp"
          accept="image/png,image/gif,image/jpeg,image/webp"
          name="avatar"
          :post-action="postAction"
          :drop="!edit"
          v-model="files"
          @input-filter="inputFilter"
          @input-file="inputFile"
          ref="upload">
          <za-button>
            上传头像
          </za-button>
        </file-upload>
      </div>
    </div>

    <div class="avatar-edit" v-show="files.length && edit">
      <div class="avatar-edit-image" v-if="files.length">
        <img ref="editImage" :src="files[0].url" />
      </div>
      <div class="text-center p-4">
        <za-button theme="error" @click.prevent="$refs.upload.clear">Cancel</za-button>
        <za-button theme="primary" @click.prevent="editSave">Save</za-button>
      </div>
    </div>
  </div>
</template>

<script>
import Cropper from 'cropperjs';
import FileUpload from 'vue-upload-component';
import './css/cropper.css';
import { apiServer, uploadAvatar } from '@/api/backend';

export default {
  components: {
    FileUpload,
  },
  data() {
    return {
      files: [],
      edit: false,
      cropper: false,
      postAction: `${apiServer}/ipfs/upload`,
    };
  },
  watch: {
    edit(value) {
      if (value) {
        this.$nextTick(function () {
          if (!this.$refs.editImage) {
            return;
          }
          const cropper = new Cropper(this.$refs.editImage, {
            aspectRatio: 1 / 1,
            viewMode: 1,
          });
          this.cropper = cropper;
        });
      } else if (this.cropper) {
        this.cropper.destroy();
        this.cropper = false;
      }
    },
  },
  methods: {
    async editSave() {
      this.edit = false;
      const oldFile = this.files[0];
      const binStr = atob(this.cropper.getCroppedCanvas().toDataURL(oldFile.type).split(',')[1]);
      const arr = new Uint8Array(binStr.length);
      for (let i = 0; i < binStr.length; i += 1) {
        arr[i] = binStr.charCodeAt(i);
      }
      const file = new File([arr], oldFile.name, { type: oldFile.type });
      await this.$refs.upload.update(oldFile.id, {
        file,
        type: file.type,
        size: file.size,
        active: true,
      });
    },
    async uploadAvatar(avatar) {
      await uploadAvatar({ avatar }, ({ error, response }) => {
        if (response.status !== 201 || error) {
          console.log(error);
          this.$Message.error('设置头像错误请重试');
          return;
        }
        this.$emit('setDone', false);
        this.$Message.success('设置成功');
      });
    },
    alert(message) {
      alert(message);
    },
    inputFile(newFile, oldFile, prevent) {
      if (newFile && oldFile
        && !newFile.active && oldFile.active
        && newFile.response.code === 200) {
        this.uploadAvatar(newFile.response.hash);
      }

      if (newFile && !oldFile) {
        this.$nextTick(function () {
          this.edit = true;
        });
      }
      if (!newFile && oldFile) {
        this.edit = false;
      }
    },
    inputFilter(newFile, oldFile, prevent) {
      // 限定最大字节
      if (newFile.size >= 0 && newFile.size > 1024 * 1024 * 2) {
        this.$Message.error('图片太大请换一张');
        return prevent();
      }
      if (newFile && !oldFile) {
        if (!/\.(gif|jpg|jpeg|png|webp)$/i.test(newFile.name)) {
          this.alert('Your choice is not a picture');
          return prevent();
        }
      }
      if (newFile && (!oldFile || newFile.file !== oldFile.file)) {
        newFile.url = '';
        const URL = window.URL || window.webkitURL;
        if (URL && URL.createObjectURL) {
          newFile.url = URL.createObjectURL(newFile.file);
        }
      }
    },
  },
};
</script>
<style>
.example-avatar .avatar-upload .rounded-circle {
  width: 200px;
  height: 200px;
}
.example-avatar .text-center .btn {
  margin: 0 .5rem
}
.example-avatar .avatar-edit-image {
  max-width: 100%
}
.example-avatar .drop-active {
  top: 0;
  bottom: 0;
  right: 0;
  left: 0;
  position: fixed;
  z-index: 9999;
  opacity: .6;
  text-align: center;
  background: #000;
}
.example-avatar .drop-active h3 {
  margin: -.5em 0 0;
  position: absolute;
  top: 50%;
  left: 0;
  right: 0;
  -webkit-transform: translateY(-50%);
  -ms-transform: translateY(-50%);
  transform: translateY(-50%);
  font-size: 40px;
  color: #fff;
  padding: 0;
}
/* modal 层级太高 */
.za-modal {
  z-index: 99;
}
</style>
