<template>
    <div class="pic-create">
        <div class="pic-tool">
            <tool-list :toolList="tList" v-if="edit" @toolListClick="onToollistClick"></tool-list>
            <div v-else>
                <y-button text="上传" :width="100"  :showIcon="false" @click="onUploadClick"></y-button>
            </div>
        </div>
       
        <context-menu   ref="contextMenu"  
                        :contextList="cList" v-show="isShowContextMenu" 
                        tabindex="3" :x="contextMenuPointX" 
                        :y="contextMenuPointY"
                        @menuItemClick="onMenuItemClick"
        ></context-menu>
        <div class="pic-grid" @contextmenu.prevent="onContextMenu" tabindex="2" @click="onPicGridClick">
            <grid-list @itemClick="onPicItemClick"></grid-list>
            <div style="height: 12vh;"></div>
        </div>
        <img-preview v-model="isShowPreview" :appendToBody="true" :previewIndex="showPreviewIndex" @closePreview="onClosePreview"></img-preview>
        <y-dialog title="上传文件" :isShow="showUploadDialog" @close="onUploadClose" theme="white">
            <el-upload
                :before-upload="beforeUpload"
                class="upload-img"
                drag
                action="/"
                multiple>
                <i class="el-icon-upload"></i>
                <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
                <div class="el-upload__tip" slot="tip">只能上传jpg/png文件，且不超过500kb</div>
            </el-upload>
        </y-dialog>
    </div>
</template>

<script>
import {mapState} from 'vuex'
import ToolList from '@/components/toollist/toollist'
import ContextMenu from '@/components/context-menu/context-menu'
import GridList from '@/components/gridlist/gridlist'
import ImgPreview from '@/components/img-preview/img-preview'
export default {
    data(){
        return {
            isShowContextMenu: false,
            isShowPreview: false,
            showPreviewIndex: 0,
            contextMenuPointX: 0,
            contextMenuPointY: 0,
            showUploadDialog: false,
            tList:[
                {
                    title: '删除',
                    iconClass: 'el-icon-delete'
                },
                {
                    title: '全选',
                    iconClass: 'el-icon-check'
                },
                {
                    title: '反选',
                    iconClass: 'el-icon-d-caret'
                },
                {
                    title: '全部取消',
                    iconClass: 'el-icon-close'
                }
            ],
            cList:[
                {
                    title: '上传',
                    iconClass: 'el-icon-upload'
                },
                {
                    title: '刷新',
                    iconClass: 'el-icon-refresh-left',
                    disabled: false
                },
                {
                    title: '排序',
                    iconClass: 'el-icon-sort',
                    disabled: false
                }],
            gList: []
        }
    },
    computed:{
        edit(){
            return !this.imgList.every((t) => {return !t.active})
        },
        ...mapState({
            imgList: state => state.image.imgList
        })
    },
    methods:{
        onToollistClick(res){
            if(res.title === '全部取消'){
                this.imgList.map(i => {
                    i.active = false
                })
            }else if(res.title === '全选'){
                 this.imgList.map(i => {
                    i.active = true
                })
            }else if(res.title === '反选'){
                 this.imgList.map(i => {
                    i.active = ! i.active
                })
            }else if(res.title === '删除'){
                let data = this.imgList.filter(i => {
                    return i.active
                }).map(k => {
                    return k._id
                })
                this.$axios.post('/person/delImages', data).then(res=>{
                    if(res.code === 200){
                        data.forEach(f => {
                            let index = this.imgList.findIndex(i => {
                                return i._id === f
                            })
                            this.imgList.splice(index, 1)
                        })
                        this.$store.dispatch('setImgList', this.imgList)
                        this.$msgbox({
                            title: '提示',
                            message: '删除成功',
                            iconClass: 'el-icon-success'
                        })
                    }
                })
                .catch((e)=>{
                    console.log(e.message)
                })
            }
        },
        onUploadClick(){
            this.showUploadDialog = true
        },
        onPicItemClick(id){
            this.showPreviewIndex = id - 1
            this.isShowPreview = true
        },
        onClosePreview(){
            this.showPreviewIndex=0
            this.isShowPreview = false
        },
        onContextMenu(e){
            if(!(e.target.className === 'pic-grid' || e.target.className === 'gridlist')) {
                return false
            }
            else{
                this.contextMenuPointX = e.offsetX
                this.contextMenuPointY = e.offsetY
                this.isShowContextMenu = true
                return true
            }
        },
        onPicGridClick(){
            this.isShowContextMenu = false
        },
        onMenuItemClick(data){
            if(data.title === '上传'){
                this.showUploadDialog = true
            }
             this.isShowContextMenu = false
        },
        onUploadClose(){
            this.showUploadDialog = false
        },
         beforeUpload(file){
            if(file.size>1*1024*1024){
                this.$message.error('psd文件不能超过1M')
                return
            }
            let temp1 = file.name.split('.')
            let temp= temp1[temp1.length-1]
            if(!['jpg','png','gif'].includes(temp)){
                this.$message.error('请上传jpg/png/gif文件')
                return false
            }
            this.uploadPsd(file)
            return false
        },
        uploadPsd(file){
            let params=new FormData()
            params.append('file',file)
            this.$axios.post('/person/uploadImage', params).then(res=>{
                if(res.code === 200){
                    let ar = res.body
                    console.log(this.imgList)
                    this.imgList.splice(this.imgList.length, 0, {
                        _id: ar._id,
                        id: this.imgList.length+1,
                        pic: ar.url,
                        active: false
                    })
                    this.showUploadDialog = false
                    this.$msgbox({
                        title: '提示',
                        message: '上传成功',
                        iconClass: 'el-icon-success'
                    })
                }
            }).catch(()=>{
            })
        }
    },
    mounted(){
            this.$axios.get('/person/images').then(res=>{
                if(res.code === 200){
                    let arr = res.body
                    let temp = []
                    arr.map((item, index)=>{
                        _id: item._id,
                        item.id = index + 1
                        item.pic = item.url
                        item.active = false
                        temp.push(item)
                    })
                    this.$store.dispatch('setImgList', temp||[])
                }
            })
        
    },
    components:{
        ToolList,
        ContextMenu,
        GridList,
        ImgPreview
    }
}
</script>

<style lang="stylus">
.pic-create{
    position: relative;
    padding: 10px 20px;
    overflow: hidden;
    .pic-tool{
        display:flex;
        width: 100px;
        height : 60px;
    }
    .pic-grid{
        height: 100%;
        overflow: auto;
    }
}
.upload-img{
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}
</style>style