<template>
    <div class="i-table-no-border">
        <!-- 搜索 -->
        <search-form
            style="border-bottom: 1px solid #eee"
            ref="searchForm"
            :show-multi-action="true"
            :multi-actions="multiActions"
            :show-export="false"
            :show-import="true"
            :show-create="false"
            :base-search-form="baseSearchForm"
            :advanced-search-form="advancedSearchForm"
            @on-import="handleShowImport"
            @on-create-form="handleOpenUpdateCreate"
            @on-search="searchData"
            @on-reset="getData"
            @on-show-recyle="handleShowRecyle"
            @on-multi-del="handleMultiDel"
        />
        <div class="ivu-mt">
            <RadioGroup v-model="modeType" type="button" @on-change="handleChangeMode">
                <Radio label="list"><Icon type="md-menu" /></Radio>
                <Radio label="thumb"><Icon type="md-apps" /></Radio>
            </RadioGroup>
            <div style="position: relative; height: 100px;" v-if="loading">
                <Spin fix size="large"></Spin>
            </div>
            <empty v-else-if="limitData.length === 0" />
            <div v-else>
                <div v-if="modeType === 'list'">
                    <Table
                        ref="table"
                        :columns="columns"
                        :data="limitData"
                        :loading="loading"
                        class="ivu-mt"
                        @on-sort-change="handleSortChange"
                        @on-filter-change="handleFilterChange"
                        @on-select="handleSelect"
                        @on-select-cancel="handleSelectCancel"
                        @on-select-all="handleSelectAll"
                        @on-select-all-cancel="handleSelectAllCancel"
                    >
                        <template slot-scope="{ row }" slot="attach_name">
                            <List>
                                <ListItem>
                                    <ListItemMeta
                                        :avatar="row.attach_full_url"
                                        :title="row.attach_name" :description="row.attach_origin_name" />
                                </ListItem>
                            </List>
                        </template>
                        <template slot-scope="{ row }" slot="attach_size">
                            大小：{{ parseFloat(row.attach_size / 1000).toFixed(2) > 1024 ? parseFloat(row.attach_size / 1000 / 1000).toFixed(2) + 'MB' : parseFloat(row.attach_size / 1000).toFixed(2) + 'KB' }}<br>
                            类型：{{ row.attach_minetype }}
                        </template>
                        <template slot-scope="{ row }" slot="attach_type">
                            <Tag v-if="row.attach_type === 1" color="primary">图片</Tag>
                            <Tag v-if="row.attach_type === 2" color="success">视频</Tag>
                            <Tag v-if="row.attach_type === 3" color="error">其他</Tag>
                        </template>
                        <template slot-scope="{ row }" slot="status">
                            <Badge v-if="row.status === 0" status="default" text="禁用" />
                            <Badge v-if="row.status === 1" status="processing" text="正常" />
                        </template>
                        <template slot-scope="{ row, index }" slot="action">
                            <a @click="handleShow(index)">查看</a>
                            <Divider type="vertical" />
                            <a @click="handleOpenUpdateCreate(true, row.id)">编辑</a>
                            <Divider type="vertical" />
                            <a @click="handleRemove(index)">删除</a>
                        </template>
                    </Table>
                </div>
                <div v-else>
                    <Row :gutter="24" class="ivu-mt search-search-projects">
                        <Col :xxl="3" :xl="6" :lg="6" :md="12" :sm="12" :xs="24" v-for="(item, index) in limitData" :key="index" class="ivu-mb">
                            <Card :bordered="bordered" :padding="0" class="search-search-projects-item">
                                <img class="search-search-projects-item-cover" v-if="item.attach_type === 1" :src="item.attach_full_url" >
                                <video class="search-search-projects-item-cover" v-if="item.attach_type === 2" muted autoplay>
                                    <source :src="item.attach_full_url" type="video/mp4">
                                    Your browser does not support the video tag.
                                </video>
                                <div class="search-search-projects-item-extra">
                                    <Time :time="item.created_at" type="datetime" />
                                </div>
                                <Divider class="ivu-mb-8 ivu-mt-8" />
                                <Row class="ivu-text-center ivu-pb-8">
                                    <Col span="12" class="ivu-br">
                                        <Tooltip placement="top" content="删除图片">
                                            <Button @click="handleRemove(index)" icon="md-trash" type="text" size="large" />
                                        </Tooltip>
                                    </Col>
                                    <Col span="12">
                                        <Tooltip placement="top" content="查看">
                                            <Button icon="md-eye" type="text" size="large" />
                                        </Tooltip>
                                    </Col>
                                </Row>
                            </Card>
                        </Col>
                    </Row>
                </div>
                <div class="ivu-mt ivu-text-right" slot="footer">
                    <Page :total="total" show-total :current.sync="current" show-elevator show-sizer :page-size="size"
                          @on-change="handleChange"
                          @on-page-size-change="handlePageSizeChange"/>
                </div>
            </div>
        </div>
        <!-- 创建编辑 -->
        <create-form ref="createForm" @on-create-form="handleOpenUpdateCreate" @on-ok="getData" />
        <!-- 上传图片 -->
        <Modal
            v-model="uploadFormSeen"
            title="选择图片"
            width="800"
            :mask-closable="false"
            @on-ok="handleOk"
            @on-cancel="handleCancel">
            <upload-form ref="uploadForm" @on-success="handleSuccess" @on-show-image="handleShowImage" :multiple="false" />
        </Modal>
    </div>
</template>
<script>
    import { AttachmentIndex, AttachmentSearch, AttachmentDelete } from '@/api/attachment';
    import SearchForm from '@/components/searchform';
    import UploadForm from '@/components/uploadform';
    import CreateForm from './create-form';
    import Empty from '@/components/common/empty';
  
    export default {
        components: {
            SearchForm,
            CreateForm,
            Empty,
            UploadForm
        },
        data () {
            return {
                modeType: 'list',
                columns: [
                    {
                        type: 'selection',
                        width: 80,
                        align: 'left'
                    },
                    {
                        title: 'ID',
                        width: 80,
                        key: 'id',
                    },
                    {
                        title: '附件信息',
                        key: 'attach_name',
                        minWidth: 300,
                        slot: 'attach_name',
                    },
                    {
                        title: '附件参数',
                        key: 'attach_size',
                        width: 200,
                        slot: 'attach_size',
                    },
                    {
                        title: '类型',
                        key: 'attach_type',
                        minWidth: 80,
                        slot: 'attach_type',
                    },
                    {
                        title: '添加时间',
                        key: 'created_at',
                        width: 180,
                        sortable: 'custom'
                    },
                    {
                        title: '操作',
                        slot: 'action',
                        align: 'center',
                        width: 200
                    }
                ],
                bordered: true,
                loading: false,
                list: [],
                selectedData: [],
                current: 1,
                total: 0,
                size: 10,
                sortType: 'normal', // 当前排序类型，可选值为：normal（默认） || asc（升序）|| desc（降序）,
                sortColumns: '',
                filterType: undefined,
                searchForm: {},
                baseSearchForm: {
                    type: 'id',
                    keyword: '',
                    options: [
                        {
                            name: 'ID',
                            value: 'id'
                        },
                        {
                            name: '附件名称',
                            value: 'attach_name'
                        },
                        {
                            name: '附件原名称',
                            value: 'attach_origin_name'
                        }
                    ]
                },
                advancedSearchForm: [
                    {
                        label_name: '附件类型：',
                        label_prop: 'attach_type',
                        ele_value: '',
                        ele_type: 'select',
                        options: [
                            {
                                value: '1',
                                name: '单图'
                            },
                            {
                                value: '2',
                                name: '视频'
                            },
                            {
                                value: '3',
                                name: '其他'
                            }
                        ],
                    },
                    {
                        label_name: '创建时间：',
                        label_prop: 'created_at',
                        ele_value: '',
                        ele_type: 'daterange',
                        options: {},
                    }
                ],
                multiActions: [
                    {
                        name: '查看回收站',
                        value: 'on-show-recyle'
                    },
                    {
                        name: '批量删除',
                        value: 'on-multi-del'
                    }
                ],
                uploadFormSeen: false
            }
        },
        computed: {
            limitData () {
                let data = [...this.list];

                // 动态计算排序类型
                const sortColumns = this.sortColumns;
                if (this.sortType === 'asc') {
                    data = data.sort((a, b) => {
                        return a[sortColumns] > b[sortColumns] ? 1 : -1;
                    });
                }
                if (this.sortType === 'desc') {
                    data = data.sort((a, b) => {
                        return a[sortColumns] < b[sortColumns] ? 1 : -1;
                    });
                }

                // 动态计算过滤类型
                if (this.filterType && this.filterType.length) {
                    data = data.filter(item => {
                        return this.filterType.indexOf(item.status) >= 0;
                    });
                }

                // 判断是否有选中的数据
                const selectedNames = this.selectedData.map(item => item.id);
                data.map(item => {
                    item._checked = selectedNames.indexOf(item.id) >= 0;
                    return item;
                });

                return data;
            },
            // 因为要动态计算总页数，所以还需要一个计算属性来返回最终给 Table 的 data
            dataWithPage () {
                const data = this.limitData;
                const start = this.current * this.size - this.size;
                const end = start + this.size;
                return [...data].slice(start, end);
            }
        },
        methods: {
            getData () {
                this.loading = true;
                AttachmentIndex({
                    page: this.current,
                    limit: this.size
                }).then(async res => {
                    this.list = res.data;
                    this.total = res.total;
                }).finally(() => {
                    this.loading = false;
                });
            },
            searchData (searchForm) {
                this.list = [];
                this.searchForm = searchForm;
                this.loading = true;
                AttachmentSearch({
                    page: this.current,
                    search: searchForm,
                    limit: this.size
                }).then(async res => {
                    this.list = res.data;
                    this.total = res.total;
                }).finally(() => {
                    this.loading = false;
                });
            },
            // 点击排序按钮时触发
            handleSortChange ({ key, order }) {
                // 将排序保存到数据
                this.sortColumns = key;
                this.sortType = order;
                this.current = 1;
            },
            // 过滤条件改变时触发
            handleFilterChange () {
                // 从第一页开始
                this.current = 1;
            },
            // 选中一项，将数据添加至已选项中
            handleSelect (selection, row) {
                this.selectedData.push(row);
            },
            // 取消选中一项，将取消的数据从已选项中删除
            handleSelectCancel (selection, row) {
                const index = this.selectedData.findIndex(item => item.id === row.id);
                this.selectedData.splice(index, 1);
            },
            // 当前页全选时，判断已选数据是否存在，不存在则添加
            handleSelectAll (selection) {
                selection.forEach(item => {
                    if (this.selectedData.findIndex(i => i.id === item.id) < 0) {
                        this.selectedData.push(item);
                    }
                });
            },
            // 取消当前页全选时，将当前页的数据（即 dataWithPage）从已选项中删除
            handleSelectAllCancel () {
                const selection = this.dataWithPage;
                selection.forEach(item => {
                    const index = this.selectedData.findIndex(i => i.id === item.id);
                    if (index >= 0) {
                        this.selectedData.splice(index, 1);
                    }
                });
            },
            // 清空所有已选项
            handleClearSelect () {
                this.selectedData = [];
            },
            handleClickItem (name) {
                if (name === 'delete') {
                    this.$Modal.confirm({
                        title: '删除标签',
                        content: '确定批量删除标签吗？',
                        onOk: () => {
                            this.selectedData.forEach(item => {
                                const index = this.list.findIndex(i => i.id === item.id);
                                if (index >= 0) {
                                    this.list.splice(index, 1);
                                }
                            });
                            this.selectedData = [];
                        }
                    });
                }
            },
            // 编辑创建数据
            handleOpenUpdateCreate (status, updateIndex) {
                this.$refs.createForm.handleShowUpdateCreate(status, updateIndex);
            },
            handleRemove (index) {
                this.updateIndex = index;
                this.$Modal.confirm({
                    title: '删除提示',
                    content: '确定删除该记录吗？',
                    onOk: () => {
                        AttachmentDelete({
                            id: this.list[index].id
                        }).then(res => {
                            this.$Message.success('删除成功');
                            this.current = 1;
                            this.getData();
                        }).finally(() => {
                        });
                    }
                });
            },
            handleChange (page) {
                this.current = page;
                if (this.searchForm) {
                    this.searchData(this.searchForm);
                } else {
                    this.getData();
                }
            },
            handlePageSizeChange (size) {
                this.current = 1;
                this.size = size;
                if (this.searchForm) {
                    this.searchData(this.searchForm);
                } else {
                    this.getData();
                }
            },
            handleShowImport () {
                this.uploadFormSeen = true;
            },
            handleOk () {
                console.log('OK')
            },
            handleCancel () {
                this.uploadFormSeen = false;
            },
            handleSuccess () {
                console.log('OK')
            },
            handleShowImage () {
                console.log('OK')
            },
            handleShowRecyle () {
                console.log('recyle')
            },
            handleChangeMode (value) {
                if (value === 'thumb') {
                    this.size = 24;
                } else {
                    this.size = 10;
                }
                this.getData();
                console.log(value)
            },
            handleMultiDel () {
                if (this.selectedData.length === 0) {
                    this.$Message.error('请选择列表');
                    return false;
                }
                this.$Modal.confirm({
                    title: '删除提示',
                    content: '确定删除该记录吗？',
                    onOk: () => {
                        AttachmentDelete({
                            id: this.list[index].id
                        }).then(res => {
                            this.$Message.success('删除成功');
                            this.current = 1;
                            this.getData();
                        }).finally(() => {
                        });
                    }
                });
            },
            handleShow () {
                console.log('show')
            }
        }
    }
</script>
<style lang="less" scoped>
    .search-search-projects {
        .ivu-mb {
            height: 250px;
        }
        &-item {
            &-cover {
                width: 100%;
                height: 150px;
                border-radius: 4px 4px 0 0;
            }
            &-desc {
                display: -webkit-box;
                height: 40px;
                -webkit-line-clamp: 2;
                -webkit-box-orient: vertical;
                overflow: hidden;
            }
            &-extra {
                text-align: center;
                span {
                    display: inline-block;
                    color: #808695;
                    vertical-align: middle;
                }
            }
        }
    }
    .ivu-avatar-image {
        border-radius: 4px !important;
    }
</style>
