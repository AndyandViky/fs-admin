<style lang="less">
    @import '../../styles/common.less';
    @import '../../styles/table.less';
</style>

<template>
    <div>
        <Row>
            <Card>
                <Row>
                    <Col span="18" style="min-height: 250px;">
                        <Table :data="tableData" :columns="visitorRecordColums" stripe ref="tableExcel"></Table>
                    </Col>
                    <Col span="6" class="padding-left-20">
                        <div class="margin-bottom-10">
                            <span>输入文件名：</span>
                            <Input v-model="imageName" icon="document" placeholder="请输入图片名" style="width: 190px"/>
                        </div>
                        <Button type="primary" @click="exportImage">导出表格为图片</Button>
                        <div class="show-image margin-top-20">
                            <img id="exportedImage" />
                        </div>
                    </Col>
                    <Col span='6' class="padding-left-10">
                        <div class="margin-top-10 margin-left-10">
                            <span>输入文件名：</span>
                            <Input v-model="excelFileName" icon="document" placeholder="请输入文件名" style="width: 190px" />
                        </div>
                        <div class="margin-left-10 margin-top-20">
                            <a id="hrefToExportTable" style="postion: absolute;left: -10px;top: -10px;width: 0px;height: 0px;"></a>
                            <Button type="primary" size="large" @click="exportExcel">下载表格数据</Button>
                        </div>
                    </Col>
                </Row>
                <center class="margin-top-10"><Page :total="total" @on-page-size-change="pageSizeChange" show-sizer @on-change="changePage"></Page></center>
            </Card>
        </Row>
    </div>
</template>

<script>
import html2canvas from 'html2canvas';
import table2excel from '@/libs/table2excel.js';
import { visitorLogColums } from '@/util/table-columns.js'
import { User } from '@/api'
export default {
    name: 'table-to-image',
    data () {
        return {
            tableData: [],
            imageName: '',
            extentionTime: 1,
            excelFileName: '',
            tableExcel: {},
            visitorRecordColums: [],
            total: 0,
            currentPage: 1,
            pageSize: 10,
        };
    },
    async created() {
        this.getData()
    },
    methods: {
        async getData() {
            const userId = this.$route.params.id;
            const data = await User.getVisitorRecord({
                pageNo: this.currentPage,
                pageSize: this.pageSize,
                userId,
            });
            this.tableData = data.datas;
            console.log(this.tableData);
            for(const item of this.tableData) {
                if(item.people) {
                    item.name = item.people.name;
                    item.belong = item.people.house_number;
                    item.adress = '幸福花园小区';
                }
            }
            this.total = data.total;
            this.visitorRecordColums = visitorLogColums(this, this.tableData)
        },
        exportImage () {
            let vm = this;
            let table = this.$refs.tableExcel.$el;
            /* 这部分代码用来解决生成的图片不清晰的问题 */
            let tableWidth = table.offsetWidth;
            let tableHeight = table.offsetHeight;
            let canvas = document.createElement('canvas');
            canvas.width = tableWidth * 2;
            canvas.height = tableHeight * 2;
            canvas.style.width = tableWidth + 'px';
            canvas.style.height = tableHeight + 'px';
            document.body.appendChild(canvas);
            var context = canvas.getContext('2d');
            context.scale(2, 2);
            /* 这部分代码用来解决生成的图片不清晰的问题 */
            html2canvas(table, {
                // canvas: canvas,
                onrendered (image) {
                    var url = image.toDataURL();
                    document.getElementById('exportedImage').src = url;
                    let a = document.createElement('a');
                    a.href = url;
                    a.download = vm.imageName ? vm.imageName : '未命名';
                    document.body.appendChild(a);
                    a.click();
                    document.body.removeChild(a);
                    // document.body.removeChild(canvas);
                }
            });
        },
        exportExcel () {
            table2excel.transform(this.$refs.tableExcel, 'hrefToExportTable', this.excelFileName);
        },
        handleExtension(val, index) {
            this.$Modal.confirm({
                onOk: () => {
                    if (this.extentionTime <= 0 || this.extentionTime > 30) {
                        this.$Message.error('输入天数必须大于0 小于 30');
                        return;
                    }
                    User.addVisiteTime({
                        deadline: this.extentionTime,
                        recordId: this.tableData[index].id
                    }).then(result => {
                        if(val.status === 2) {
                            val.status = 1;
                        }
                        this.$Message.info('延期成功');
                    });
                },  
                render: (h) => {
                    return h('Input', {
                        props: {
                            value: this.extentionTime,
                            autofocus: true,
                            placeholder: '请输入延期天数, 最大30天, 最少1天...'
                        },
                        on: {
                            input: (val) => {
                                this.extentionTime = val;
                            },
                            ok: () => {
                                this.$Message.info("延期成功");
                            }
                        }
                    })
                }
            })
        },
        changePage(page) {
            this.currentPage = page;
            this.getData();
        },
        pageSizeChange(size) {
            this.pageSize = size;
        }
    }
};
</script>
