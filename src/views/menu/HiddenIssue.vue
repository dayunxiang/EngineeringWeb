<template>
    <section>
        <CommonCRUD
            :formColumns="formColumns"
            apiRoot="/identity/hiddenIssue" :queryFormColumns="queryFormColumns"
            :columns="columns" :addBtnVis="false" :editBtnVis="false" :delBtnVis="false" ref="table">
            <template slot="header-btn" slot-scope="slotProps">
                <el-button type="info" v-if="dealBtn" plain  class="self-btn self-deal" @click="deal(slotProps.selected)">&nbsp;</el-button>
            </template>
            <template slot="hiddenStatus" slot-scope="scope">
                <el-tag v-if="scope.row.statusName==='完成'"
                        close-transition type="success">完成</el-tag>
                <el-tag v-if="scope.row.statusName==='待处理'"
                        close-transition type="danger">待处理</el-tag>
                <el-tag v-if="scope.row.statusName==='待审核'"
                        close-transition type="warning">待审核</el-tag>
            </template>
        </CommonCRUD>.
        <el-dialog
            v-if="dialogVisible"
            :title="title"
            :visible.sync="dialogVisible"
            width="880px"
            align="left"
            :modal-append-to-body='false'
            :append-to-body="true"
            :before-close="handleClose">
            <el-form :inline="true" :model="form"  ref="form" :rules="rules" label-width="170px" class="demo-ruleForm">
                <el-form-item style="display: none">
                    <el-input v-model="form.issueId" type="hidden"></el-input>
                </el-form-item>

                <el-form-item label="图片" >
                    <CommonUpload :value="form.solveImage" @getValue="form.solveImage = $event"></CommonUpload>
                </el-form-item>
                <el-form-item label="附件" >
                    <CommonFileUpload :value="form.enclosure" @getValue="form.enclosure = $event"></CommonFileUpload>
                </el-form-item>
                <el-form-item label="完成日期" >
                    <el-date-picker  v-model="form.solveTime"
                                     type="date"
                                     value-format="yyyy-MM-dd"
                                     placeholder="选择日期">
                    </el-date-picker>
                </el-form-item>
                <el-form-item label="处理结果描述" prop="solveDes">
                    <el-input type="textarea" class="result_area" :autosize="{ minRows: 3, maxRows: 5}" v-model="form.solveDes"></el-input>
                </el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer footer-position">
                <el-button type="primary" :loading="submitLoading" @click="submit('form')">确 定</el-button>
                <el-button @click="handleClose">取 消</el-button>
            </div>
        </el-dialog>
    </section>
</template>

<script>
    import CommonCRUD from '@/components/CommonCRUD';
    import CommonUpload from '@/components/UpLoad';
    import CommonFileUpload from '@/components/FileUpLoad';
    import LookUp from '@/lookup';
    import { tansfer } from "../../lookup/transfer";
    export default {
        name: 'HiddenIssue',
        data() {
            return {
                formColumns:[],
                columns:[],
                roleCode:'',
                uploadBtn:true,
                dealBtn:true,
                dialogVisible: false,
                form:{id:'',createdAt:'',createdBy:'',issueId:'',solveImage:'',solveDes:'',solveTime:'',enclosure:''},
                submitLoading: false,
                title: '隐患处理',
                disabled: false,
                selected: [],
                rules: {
                    solveDes:[{ required: true, message: '请输入处理结果描述', trigger: 'blur' }]
                },
                queryForm:{issueId:''},
                recordsForm: {issueId:'',preNodeId:'',nextNodeId:'',actionType:'',des:''},
                queryFormColumns:[
                    {name:'userId',value:''},
                    {name:'status',value:'3dfa705b-c5ec-4e95-9838-0045022358bb'},
                    {
                        name:'issueName',
                        visible:true,
                        des:'隐患名称',
                        type:'string'
                    },
                    {
                        name:'projectId',
                        visible:true,
                        des:'工程名称',
                        type:'select',
                        options:Array
                    },
                    {
                        name:'issueGrade',
                        visible:true,
                        des:'隐患等级',
                        type:'select'
                    }
                    ]
            }
        },
        methods:{
            handleSelectOptions() {
                let  items = [
                    ['issueGrade', 'IssueType']
                ].forEach(item => {
                        this.formColumns.filter(sub => sub.name === item[0])[0].options = LookUp[item[1]]
                    }
                )
                tansfer(this.columns);
            },
            deal(val) {
                if (val.length !== 1) {
                    this.$message({
                        type: 'warning',
                        message: val.length > 1 ? '仅能选择一行记录' : '请选择一行记录'
                    });
                    return false;
                }
                if(val[0].statusName == '待审核'|| val[0].statusName == '完成' ){
                    this.$message({
                        type: 'warning',
                        message: '处理已完成'
                    });
                    return false;
                }
                this.form = {}
                if (this.$refs['form'] !== undefined) {
                    this.$refs['form'].resetFields();
                }
                this.form.issueId = val[0].id
                this.queryForm.issueId = val[0].id
                this.dialogVisible = true
             },
            submit(formName) {
                this.$refs[formName].validate((valid) => {
                    if (valid) {
                        this.submitLoading = false
                        this.$http('Post', '/identity/hiddenHandle/list',this.queryForm, false).then(
                            data => {
                                if(data.length>0){
                                    this.form.id = data[0].id
                                    this.form.createdAt = data[0].createdAt
                                    this.form.createdBy = data[0].createdBy
                                    let type = 'Put';
                                    let path = '/identity/hiddenHandle/'+data[0].id+'id';
                                    this.$http(type, path, Object.assign({}, this.form)).then((data1) => {
                                        this.$nextTick(() => {
                                            let type = 'Post'
                                            let path = '/identity/hiddenRecords/'
                                            this.recordsForm.issueId = data1.issueId
                                            this.recordsForm.actionType='HANDLE'
                                            this.recordsForm.preNodeId='3dfa705b-c5ec-4e95-9838-0045022358bb'
                                            this.recordsForm.nextNodeId='86427b26-b4c3-462c-8ce0-4992098534eb'
                                            this.recordsForm.des='隐患处理'
                                            this.$http(type,path,this.recordsForm).then(()=>{
                                                this.recordsForm = {};
                                            })
                                        })
                                        this.submitLoading = false;
                                        this.dialogVisible= false;
                                        this.$refs.table.refreshTableData();
                                        this.form = {};
                                    });
                                }else{
                                    this.$nextTick(() => {
                                        let type = 'Post';
                                        let path = '/identity/hiddenHandle/';
                                        this.$http(type, path, Object.assign({}, this.form)).then((data1) => {
                                            this.$nextTick(() => {
                                                let type = 'Post'
                                                let path = '/identity/hiddenRecords/'
                                                this.recordsForm.issueId = data1.issueId
                                                this.recordsForm.actionType='HANDLE'
                                                this.recordsForm.preNodeId='3dfa705b-c5ec-4e95-9838-0045022358bb'
                                                this.recordsForm.nextNodeId='86427b26-b4c3-462c-8ce0-4992098534eb'
                                                this.recordsForm.des='隐患处理'
                                                this.$http(type,path,this.recordsForm).then(()=>{
                                                    this.recordsForm = {};
                                                })
                                            })
                                            this.submitLoading = false;
                                            this.dialogVisible= false;
                                            this.$refs.table.refreshTableData();
                                            this.form = {};
                                        });
                                    })
                                }
                            }
                        ).catch(res => {
                            this.dialogVisible = true;
                            console.log('error submit!!');
                            return false;
                        });


                    } else {
                        this.dialogVisible = true;
                        console.log('error submit!!');
                        return false;
                    }
                });
            },
            //权限控制（列表数据）
            controlAuthority(){
                this.roleCode = JSON.parse(sessionStorage.getItem('userInfo')).roleCode;
                //上报人显示增删改查按钮
                if( this.roleCode === 'PROJECT_MANAGER'){
                    //显示自己上报的内容
                    this.queryFormColumns[0].value = JSON.parse(sessionStorage.getItem('userInfo')).id;
                }else{
                    this.queryFormColumns[0].value = null;
                }
        },
            handleClose (done) {
                this.$confirm('确认关闭？')
                    .then(_ => {
                        this.from = {};
                        this.$refs['form'].resetFields();
                        this.dialogVisible = false;
                        done();
                    })
                    .catch(_ => {});
            },
            loadDepartmentOptions() {
                this.$http('POST', 'identity/organization/list', false).then(
                    data => {
                        this.formColumns.filter( item => item.name === 'departmentId')[0].options = data.map(item => { return {value: item.id, label: item.name}});
                    }
                )
                this.$http('POST', 'identity/projectInfo/list', false).then(
                    data => {
                        this.formColumns.filter( item => item.name === 'projectId')[0].options = data.map(item => { return {value: item.id, label: item.name}});
                    }
                )
                this.$http('POST', 'identity/principal/list', false).then(
                    data => {
                        this.formColumns.filter( item => item.name === 'userId')[0].options = data.map(item => { return {value: item.id, label: item.name}});
                    }
                )
            },
            //查询select赋值
            loadSelect() {

                let  items = [
                    ['issueGrade', 'IssueType']
                ].forEach(item => {
                        this.queryFormColumns.filter(sub => sub.name === item[0])[0].options = LookUp[item[1]]
                    }
                )
                this.$http('POST', 'identity/projectInfo/list', false).then(
                    data => {
                        this.queryFormColumns.filter( item => item.name === 'projectId')[0].options = data.map(item => { return {value: item.id, label: item.name}});
                    }
                )
            }

        },
        components :{
            CommonUpload,
            CommonCRUD,
            CommonFileUpload
        },
        created () {
            this.columns = []
            this.columns.length = 0;
            let temp = JSON.parse(JSON.stringify(this.$store.state.classInfo.properties));
            let temp1 = JSON.parse(JSON.stringify(this.$store.state.classInfo.properties));
            this.formColumns = temp1
            this.columns = temp
            var columsItems1 = {slot:true,name:'hiddenStatus',des:'状态',slotName:'hiddenStatus'}
            this.columns.push(columsItems1)
            this.loadDepartmentOptions();
            this.handleSelectOptions();
            this.controlAuthority();
            this.loadSelect();
        }
    };
</script>

<style>
    .result_area .el-textarea__inner{
        width: 580px !important;
    }
</style>
