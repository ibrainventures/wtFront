<template>
    <div class="report-tree">

        <q-breadcrumbs separator="●" color="light" active-color="dark" class="navigator">
            <q-breadcrumbs-el label="HOME" to="/" />
            <!--<q-breadcrumbs-el label="Project" to="/project" />-->
        </q-breadcrumbs>

        <q-select
            v-model="select"
            :options="periodOptions"
            class="report-tree-select"/>

        <q-btn icon="file download" label="EXPORT" @click="exportExcel" class="btn-create" v-if="isAdmin" />
        <q-btn icon="code" label="USER" @click="redirectUser" class="btn-create" v-if="isAdmin" />
        <q-btn icon="code" label="PROJECT" @click="createProject" class="btn-create" v-if="showUser" />
        <q-btn icon="add" label="TASK" @click="createTask" class="btn-create" v-if="!historyTask" />
        <q-collapsible v-for="(item, index) in tableData" popup icon="layers" :label="item.project" :key="index">
            <div>
                <q-table
                    :data="item.data"
                    :columns="showUser ? columnsLeader : columns"
                    selection="single"
                    :selected.sync="item.selected"
                    :pagination.sync="paginationControl"
                    color="primary"
                    no-data-label="暂无数据"
                    table-class="task-table">
                    <template slot="top-selection" slot-scope="props">
                        <q-btn color="positive" flat icon="mode edit" label="编辑" @click="editTask(item.selected)"  />
                        <q-btn color="negative" flat delete icon="delete" label="删除" @click="deleteTask(item.selected)" />
                    </template>
                </q-table>
            </div>
        </q-collapsible>

        <q-modal v-model="createTaskModal" @hide="resetForm" :content-css="{padding: '50px', minWidth: '500px'}">
            <div v-if="!isEdit" class="q-display-1 q-mb-md">创建任务</div>
            <div v-if="isEdit" class="q-display-1 q-mb-md">编辑任务</div>
            <div>
                <q-field
                        class="form-field"
                        :error="$v.taskForm.name.$error"
                        :error-label="$v.taskForm.name.maxLength ? errMessage.requireInfo : errMessage.maxInfo">
                    <q-input float-label="任务名称"
                             @input="$v.taskForm.name.$touch"
                             v-model="taskForm.name" />
                    <!--<span class="form-group__message" v-if="!$v.taskForm.name.maxLength">任务长度不宜超过 {{$v.taskForm.name.$params.maxLength.max}} 字符.</span>-->
                </q-field>
                <q-field
                        class="form-field"
                        :error="$v.taskForm.project_id.$error"
                        :error-label="errMessage.requireInfo">
                    <q-select
                            v-model="taskForm.project_id"
                            float-label="所属项目"
                            :options="projectOptions"/>
                </q-field>
                <q-field
                        class="form-field"
                        :error="$v.taskForm.status.$error"
                        :error-label="errMessage.requireInfo">
                    <q-select
                            v-model="taskForm.status"
                            float-label="状态"
                            :options="statusOptions"/>
                </q-field>
                <q-field
                        class="form-field"
                        :error="$v.taskForm.progress.$error"
                        :error-label="errMessage.requireInfo">
                    <q-select
                            v-if="taskForm.status != 2"
                            v-model="taskForm.progress"
                            float-label="进度"
                            :options="progressOptions"/>
                </q-field>
                <q-field
                        class="form-field"
                        :error="$v.taskForm.remark.$error"
                        :error-label="errMessage.requireInfo">
                    <q-input float-label="备注"
                             @input="$v.taskForm.remark.$touch"
                             v-model="taskForm.remark" />
                </q-field>
            </div>
            <q-btn
                    :loading="loading"
                    color="primary"
                    class="btn-save"
                    @click="saveTask">
                保存
                <q-spinner-hourglass slot="loading" size="20px" />
                <span slot="loading">Loading...</span>
            </q-btn>
        </q-modal>

        <q-modal v-model="createProjectModal" :content-css="{padding: '50px', minWidth: '500px'}">
            <div class="q-display-1 q-mb-md">创建项目</div>
            <div>
                <q-field
                        class="form-field"
                        :error="$v.projectForm.name.$error"
                        error-label="Required">
                    <q-input float-label="项目名称"
                             @input="$v.projectForm.name.$touch"
                             v-model="projectForm.name" />
                </q-field>
                <q-btn
                        :loading="loadingProject"
                        color="primary"
                        class="btn-save"
                        @click="saveProject">
                    保存
                    <q-spinner-hourglass slot="loading" size="20px" />
                    <span slot="loading">Loading...</span>
                </q-btn>
            </div>
        </q-modal>
    </div>
</template>

<script>
    import {required, minLength, maxLength} from 'vuelidate/lib/validators';
    import {date} from 'quasar';
    export default {
        name: 'REPORT',
        data () {
            return {
                isDeploy: false,
                isEdit: false, // task是否处于编辑状态
                isAdmin: false, // 判断是否是超级管理员
                showUser: false, // 判断是否是二级管理员
                historyTask: false,
                loading: false,
                loadingProject: false,
                createTaskModal: false,
                createProjectModal: false,
                paginationControl: {rowsPerPage: 0, page: 1},
                weekOfYear: '',
                user: {},
                select: '2018-03-01',
                selectOptions: [
                    {label: '2018-03-01', value: '2018-03-01'},
                    {label: '2018-02-20', value: '2018-02-20'},
                    {label: '2018-02-13', value: '2018-02-13'}
                ],
                periodOptions: [],
                projectsMock: [
                    {id: 1, name: 'Portal'},
                    {id: 2, name: 'Admin'},
                    {id: 3, name: 'Mini Programs'}
                ],
                projectOptions: [],
                statusOptions: [
                    {label: '开发中', value: 0},
                    {label: '已提测', value: 1},
                    {label: '已上线', value: 2}
                ],
                progressOptions: [
                    {label: '0%', value: 0},
                    {label: '10%', value: 10},
                    {label: '20%', value: 20},
                    {label: '30%', value: 30},
                    {label: '40%', value: 40},
                    {label: '50%', value: 50},
                    {label: '60%', value: 60},
                    {label: '70%', value: 70},
                    {label: '80%', value: 80},
                    {label: '90%', value: 90},
                    {label: '100%', value: 100}
                ],
                taskForm: {
                    id: '',
                    name: '',
                    user_id: '',
                    project_id: '',
                    progress: 0,
                    status: 0,
                    remark: '',
                    period: ''
                },
                projectForm: {
                    name: ''
                },
                errMessage: {
                    requireInfo: '必填',
                    maxInfo: '任务名称不宜超过30个字符'
                },
                columns: [
                    {name: '任务名称', label: '任务名称', field: 'name', align: 'left'},
                    {name: '状态', label: '状态', field: 'statusZh'},
                    {name: '进度', label: '进度', field: 'progressPercent'}
                ],
                columnsLeader: [
                    {name: '任务名称', label: '任务名称', field: 'name', align: 'left'},
                    {name: '状态', label: '状态', field: 'statusZh'},
                    {name: '进度', label: '进度', field: 'progressPercent'},
                    {name: '责任人', label: '责任人', field: 'username'}
                ],
                tableDataMock: [{
                    project: '门户',
                    selected: [],
                    data: [
                        {
                            id: 1,
                            name: 'medadwa',
                            progress: 100,
                            status: 0
                        },{
                            id: 2,
                            name: 'medadwa2',
                            progress: 60,
                            status: 1
                        },{
                            id: 3,
                            name: 'medadwasada',
                            progress: 10,
                            status: 2
                        },{
                            id: 11,
                            name: 'medadwa',
                            progress: 100,
                            status: 0
                        },{
                            id: 12,
                            name: 'medadwa2',
                            progress: 60,
                            status: 1
                        },{
                            id: 13,
                            name: 'medadwasada',
                            progress: 10,
                            status: 2
                        },{
                            id: 21,
                            name: 'medadwa',
                            progress: 100,
                            status: 0
                        },{
                            id: 22,
                            name: 'medadwa2',
                            progress: 60,
                            status: 1
                        },{
                            id: 23,
                            name: 'medadwasada',
                            progress: 10,
                            status: 2
                        }
                    ]
                },{
                    project: '开发者中心',
                    selected: [],
                    data: [
                        {
                            id: 4,
                            name: 'medadwa',
                            progress: 100,
                            status: 0
                        },{
                            id: 5,
                            name: 'medadwa2',
                            progress: 60,
                            status: 1
                        },{
                            id: 6,
                            name: 'medadwasada',
                            progress: 10,
                            status: 2
                        },
                    ]
                }],
                tableData: [{
                    project: '暂无数据',
                    selected: [],
                    data: []
                }]
            }
        },
        validations: {
            taskForm: {
                name: {required, maxLength: maxLength(30)},
                project_id: {required},
                progress: {required},
                status: {required},
                remark: {}
            },
            projectForm: {
                name: {required}
            }
        },
        created () {
            this.initFormData();
        },
        watch: {
            select: function () {
                this.weekOfYear = this.select;
                this.getReportData();
            }
        },
        methods: {
            initFormData () {
                const _this = this;
                _this.user = JSON.parse(localStorage.getItem('user'));
                _this.isAdmin = _this.user.role === 0;
                _this.showUser = _this.user.role !== 2;
                _this.taskForm.user_id = _this.user._id;
                _this.checkFinished();
                _this.getWeekOfYear();
                _this.renderPeriods();
//                _this.getReportData();
                _this.getProjectsList();
            },
            checkFinished () {
                const _this = this;
                _this.$axios.post('/weeklyreportapi/isFinished', {user_id: _this.user._id}).then((res) => {
                    if (res.data.code === 0) {
                        _this.$q.notify({
                            message: res.data.message,
                            timeout: 3000,
                            type: 'positive',
                            position: 'top'
                        });
                        _this.getReportData();
                    }
                }).catch((error) => {
                    _this.handleError(error);
                });
            },
            renderPeriods () {
                const _this = this;
                for (let i = parseInt(_this.weekOfYear); i > 9; i--) {
                    _this.periodOptions.push({
                        label: (_this.isAdmin ? '' : _this.user.name + ' ') + '第'+ i + '期周报',
                        value: i
                    });
                }
                _this.select = parseInt(_this.weekOfYear);
            },
            getProjectsList () {
                const _this = this;
                _this.projectOptions = [];
                _this.$axios.post('/weeklyreportapi/getProjectListByTeam', {team: _this.user.team}).then((res) => {
                    if (res.data.code === 0) {
                        let data = res.data.data;
                        for (let i = 0, size = data.length; i < size; i++) {
                            _this.projectOptions.push({
                                label: data[i].name,
                                value: data[i]._id
                            });
                        }
                    }
                }).catch((error) => {
                    _this.handleError(error);
                });
            },
            getReportData () {
                const _this = this;
                let queryParams = {
                    period: _this.weekOfYear,
                    username: _this.user.name,
                    userrole: _this.user.role,
                    userid: _this.user._id
                };
                _this.$axios.post('/weeklyreportapi/getTaskListByPeriod', queryParams).then((res) => {
                    if (res.data.code === 0) {
                        if (res.data.data.length > 0) {
                            _this.tableData = res.data.data;
                        } else if (res.data.data.length === 0) {
                            _this.$q.notify({
                                message: '第'+ _this.weekOfYear + '期周报暂无数据,会自动跳转到最新一期',
                                timeout: 3000,
                                type: 'positive',
                                position: 'top'
                            });
                            _this.getWeekOfYear();
                            _this.select = parseInt(_this.weekOfYear);
                        }
                    }
                }).catch((error) => {
                    _this.handleError(error);
                });
            },
            getWeekOfYear () {
                let tempWeekOfYear = date.formatDate(Date.now(), 'w');
                this.weekOfYear = date.formatDate(Date.now(), 'd') === 0 ? tempWeekOfYear + 1 : tempWeekOfYear;
            },
            createTask () {
                this.createTaskModal = true;
            },
            saveTask () {
                const _this = this;
                _this.$v.taskForm.$touch();
                if (_this.$v.taskForm.$error) {
                    return;
                }
                // 如果状态是已上线,那么进度默认为100
                if (_this.taskForm.status === 2) {
                    _this.taskForm.progress = 100;
                }
                _this.loading = true;
                _this.$axios.post(_this.isEdit ? '/weeklyreportapi/task/edit' : '/weeklyreportapi/task/add', _this.taskForm).then((res) => {
                    if (res.data.code === 0) {
                        _this.getReportData();
                        _this.$q.notify({
                            message: res.data.message,
                            timeout: 3000,
                            type: 'positive',
                            position: 'top'
                        });
                        setTimeout(()=>{
                            _this.loading = false;
                            _this.createTaskModal = false;
                            _this.isEdit = false;
                            _this.resetForm();
                        }, 1000);
                    } else {
                        _this.loading = false;
                        _this.$q.dialog({
                            title: 'Error',
                            message: res.data.message
                        });
                    }
                }).catch((error)=>{
                    _this.handleError(error);
                });
            },
            editTask (props) {
                const _this = this;
                _this.taskForm.id = props[0].id;
                _this.taskForm.name = props[0].name;
                _this.taskForm.period = props[0].period;
                _this.taskForm.progress = props[0].progress;
                _this.taskForm.project_id = props[0].project._id;
                _this.taskForm.status = parseInt(props[0].status);
                _this.taskForm.user_id = _this.user._id;
                _this.taskForm.remark = props[0].remark;
                _this.createTaskModal = true;
                _this.isEdit = true;
            },
            deleteTask (props) {
                const _this = this;
                this.$q.dialog({
                    title: '确认',
                    message: '确认删除该任务吗?',
                    ok: '删除',
                    cancel: '再考虑考虑'
                }).then(() => {
                    _this.$axios.post('/weeklyreportapi/task/del', {
                        id: props[0].id,
                        user_id: _this.user._id
                    }).then((res) => {
                        if (res.data.code === 0) {
                            _this.getReportData();
                            _this.$q.notify({
                                message: '已删除,这下真没了!',
                                timeout: 3000,
                                type: 'positive',
                                position: 'top'
                            });
                        } else {
                            _this.loading = false;
                            _this.$q.dialog({
                                title: 'Error',
                                message: res.data.message
                            });
                        }
                    }).catch((error)=>{
                        _this.handleError(error);
                    });
                }).catch(() => {
                    _this.$q.notify({
                        message: '看来你是一个很谨慎的人!',
                        timeout: 3000,
                        type: 'info',
                        position: 'top'
                    });
                })
            },
            createProject () {
//                this.createProjectModal = true;
                this.$router.push('/project');
            },
            redirectUser () {
                this.$router.push('/user');
            },
            saveProject () {
                const _this = this;
                _this.$v.projectForm.$touch();
                if (_this.$v.projectForm.$error) {
                    return;
                }
                _this.loadingProject = true;
                _this.$axios.post('/weeklyreportapi/project/add', _this.projectForm).then((res) => {
                    if (res.data.code === 0) {
                        _this.getProjectsList();
                        _this.$q.notify({
                            message: res.data.message,
                            timeout: 3000,
                            type: 'positive',
                            position: 'top'
                        });
                        setTimeout(()=>{
                            _this.loadingProject = false;
                            _this.createProjectModal = false;
                            _this.projectForm.name = '';
                        }, 1000);
                    } else {
                        _this.loading = false;
                        _this.$q.dialog({
                            title: 'Error',
                            message: res.data.message
                        });
                    }
                }).catch((error)=>{
                    _this.handleError(error);
                });
            },
            exportExcel () {
                this.$axios.post('/weeklyreportapi/export', {period: this.weekOfYear}).then((res) => {
                    if (res.data.code === 0) {
//                        window.open('https://www.ioteams.com/weeklyreportapi/'+res.data.data.url);
                        window.open('http://localhost:22230/index.html#/dist/spa-mat/statics/'+res.data.data.url);
                    }
                }).catch((err)=>{
                    this.$q.notify({
                        message: err.response.data.message,
                        timeout: 3000,
                        type: 'info',
                        position: 'top'
                    });
                });
            },
            resetForm () {
                const _this = this;
                _this.taskForm.id = '';
                _this.taskForm.name = '';
//                _this.taskForm.user_id = '';
                _this.taskForm.project_id = '';
                _this.taskForm.progress = 0;
                _this.taskForm.status = 0;
                _this.taskForm.remark = '';
                _this.taskForm.create_date = '';
                _this.taskForm.period = '';
                _this.isEdit = false;
            },
            handleError (error) {
                let isExpired = error.response.data.error === 'jwt expired';
                if (error.response.status !== 500) {
                    this.$q.notify({
                        message: isExpired ? 'token已过期,重新登录' : error.response.data.error,
                        timeout: 3000,
                        type: 'negative',
                        position: 'top',
                        actions: isExpired ? [{
                            label: '登录',
                            handler: () => {
                                this.$router.push('/login');
                            }
                        }] : null
                    });

                    if (isExpired) {
                        setTimeout(()=> {
                            this.$router.push('/login');
                        }, 1000);
                    }
                } else {
                    this.loading = false;
                    this.loadingProject = false;
                    this.$q.dialog({
                        title: error.response.status + '',
                        message: error.response.data.message
                    });
                }
            }
        }
    }
</script>

<style lang="less">
    .report-tree {
        position:absolute;
        width:60%;
        height:50%;
        top:50%;
        left:50%;
        transform:translate(-50%,-50%);
    }
    .navigator {
        position: absolute;
        left: 6px;
        top: -60px;
        margin: 0;
    }
    .report-tree-select {
        position: absolute;
        right: 15px;
        top: -10px;
        margin: 0;
    }
    .btn-create {
        position: relative;
        top: -15px;
        left: 15px;
        display: inline-block;
    }
    .btn-save {
        margin-top: 10px;
    }
    .form-field {
        margin: 12px 0;
    }
    .q-table-container {
        -webkit-box-shadow: none;
        -moz-box-shadow: none;
        box-shadow: none;
    }
    .task-table {
        border: none;
    }
    .q-table-top {
        position: absolute;
        top: -45px;
        right: 50px;
        min-height: 0;
        padding: 0;
    }
</style>