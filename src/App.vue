<script setup>
import { ref, watch, toRefs, toValue } from 'vue'
import { ElMessage } from 'element-plus';
import useStore from "@/store/index";
import { getUserInfo, takeAllBooks, takeDoc, createSummary } from "@/axios/api";
import matchSummary from "@/utils/matchSummaryGroup";
import InitDialog from "./view/InitDialog.vue";
import { generateGroup, mergeSummary, sortTime ,getMergeTitle} from "@/utils/tools.js";
import TreeContent from "@/view/TreeContent.vue";
// 定义一个全局状态
const { store, updateUserInfo, setSelectBookInfo } = useStore()

const awaitMergeDocs = ref([]);
// 合并目标文档
const targetDoc = ref([]);

//!获取对应的知识库列表
const books = ref([]);
// 选择的书籍信息
const selectBook = ref(null);

const { userInfo } = toRefs(store)

const treeContent = ref(null);

const isLoading = ref(false);

const rest = ()=>{
    treeContent.value.restCheckedTreeNode();
}

//! 获取用户信息
async function initializeUserAndBooks() {

    try {
        const userInfoResponse = await getUserInfo();

        updateUserInfo(userInfoResponse);
        
        const bookList = await takeAllBooks(userInfoResponse.id);

        books.value = bookList;
        // 设置选择的默认值
        selectBook.value = 24552766;
    } catch (error) {
        console.error('初始化用户和书籍信息时发生错误:', error);
    }
}

// 选择发生变化得到选择的select
watch(selectBook, () => {

    if (selectBook.value !== "") {

        let result = books.value.find((item) => item.id === selectBook.value)

        setSelectBookInfo(result);
    }
})

//合并逻辑
async function mergeDoc() {
    const choice = targetDoc.value.filter((item) => !item.children.length).length
    
    if (choice !== 1 
    || awaitMergeDocs.value.length === 0) {
        return ElMessage.error("文档选择不正确");
    }
    isLoading.value = true;
    // 得到所有的文章数据
    const docs = await getSelectedDocs(toValue(awaitMergeDocs));
   
    // 将文章转换为分类对象 
    const cryObj = docs.flatMap((doc) => matchSummary(doc))

    //根据最早最晚得到标题
    const title =  getMergeTitle(cryObj)
    
    //将分类对象进行分组 
    const cryGroup = generateGroup(cryObj)
    
    //对分类对象进行排序
    Object.keys(cryGroup).forEach((k) => cryGroup[k] = sortTime(cryGroup[k]))
   
    let newDocStr = mergeSummary(cryGroup)
 
    // return
    await createSummary(`/repos/${selectBook.value}/docs/${targetDoc.value[0].id}`, {
        slug: targetDoc.value[0].id,
        title: title,
        body: newDocStr
    })

    ElMessage.success("合并成功!")

    isLoading.value = false;
}
// 获取所有的文章数据
function getSelectedDocs(docs) {

    const urls = docs.map((item) => `/repos/${selectBook.value}/docs/${item.id}`);

    return Promise.all(urls.map((u) => takeDoc(u)))
}

const setAwaitMergeDocs = (val)=>{

    awaitMergeDocs.value = val;
}

const setTargetDoc = (val)=>{
    targetDoc.value = val;
}

</script>

<template>
    <div class="common-layout" v-loading.fullscreen.lock="isLoading">
        <el-container class="container">
            <el-header class="header">
                <el-row style="margin-top: 15px;">
                    <el-col :span="6">
                        <el-select placeholder="选择知识库" v-model="selectBook">
                            <el-option v-for="item in books" :key="item.id" :label="item.name" :value="item.id" />
                        </el-select>
                    </el-col>
                    <el-col :offset="2" :span="4">
                        <el-text>hello {{ userInfo.name }}</el-text>
                    </el-col>
                    <el-col :offset="3" :span="2">
                        <el-button type="primary" @click="mergeDoc">
                            merge
                        </el-button>
                    </el-col>
                    <el-col :offset="3" :span="3">
                        <el-button type="primary" @click="rest">
                            rest
                        </el-button>
                    </el-col>
                </el-row>
            </el-header>
            <el-container>
                <el-main>
                    <TreeContent ref="treeContent" :set-await-merge-docs="setAwaitMergeDocs" :set-target-doc="setTargetDoc"/>
                </el-main>
            </el-container>
        </el-container>
        <InitDialog :initializeUserAndBooks="initializeUserAndBooks"></InitDialog>
    </div>
</template>

<style scoped lang="less">
.common-layout {
    height: 80%;
    width: 70%;
    margin: 5% auto;

    .container {
        height: 100%;

        .header {
            border: 1px solid #000;
        }
    }

}

.title {
    display: flex;
    gap: 10px;
    font-size: 25px;
    margin-bottom: 10px
}

.ellipsis {
    display: inline-block;
    max-width: 200px; /* 可以根据需要调整宽度 */
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}
</style>
