<template>
  <div style="height:100%">
    <div v-if="hasOperation(['search', 'title', 'custom'])" class="top-operations">
      <div class="top-title">
        <slot name="topTitle"></slot>
      </div>
      <div class="custom" v-if="hasOperation(['custom'])">
        <a-dropdown v-model="customListDrop">
          <a-menu slot="overlay">
            <a-menu-item v-for="(column, index) in tableCustoms.filter(x => x.title && x.property!='chrName' && !x.type)" :key="index">
              <vxe-checkbox v-model="column.visible" :key="index" @change="$refs.xTable.refreshColumn()">{{ column.title }}</vxe-checkbox>
            </a-menu-item>
          </a-menu>
          <a-button type="primary" style="margin-left: 8px;padding:0 8px;"> <a-icon type="bars" /> <a-icon type="down" /> </a-button>
        </a-dropdown>
      </div>
      <div class="search" v-if="hasOperation(['search'])">
        <a-input-search type="text" v-model="searchValue" @search="handleSearch" enterButton></a-input-search>
      </div>
    </div>
    <div :class="calTableHeightClass()">
      <vxe-grid
        ref="xTable"
        :resizable="resizable"
        :border="border"
        :stripe="stripe"
        :highlight-hover-row="highlightHoverRow"
        height="100%"
        auto-resize
        :columns="tableColumns"
        :customs.sync="tableCustoms"
        :loading="loading"
        @select-change="handleSelectChange"
        @select-all="handleSelectChange"
        @radio-change="handleRadioChange"
        :show-footer="calculateMethods.length?true:false"
        :footer-method="footerMethod"
        show-overflow
        show-header-overflow
        :optimization="{scrollY:{gt:20}}"
        :checkbox-config="checkboxConfig"
        :row-id="xid"
        @cell-click="doCellClick"
      ></vxe-grid>
    </div>
    <div class="bottom-operations" v-if="hasOperation(['delete']) || pager">
      <slot name="bottom">
        <a-popconfirm
          v-if="hasOperation(['delete'])"
          title="确定删除吗?"
          :visible="deleteConfirmVisible"
          @visibleChange="handleDeleteConfirmVisible"
          @confirm="handleDelete()"
          @cancel="deleteConfirmVisible=false"
        >
          <a-button class="ant-btn ant-btn-primary xtable-bottom-opt-btn">删除</a-button>
        </a-popconfirm>
        <a-button
          v-if="hasOperation(['valid'])"
          class="ant-btn ant-btn-primary xtable-bottom-opt-btn"
          @click="handleValid('1')"
        >启用</a-button>
        <a-button
          v-if="hasOperation(['invalid'])"
          class="ant-btn ant-btn-primary xtable-bottom-opt-btn"
          @click="handleValid('0')"
        >停用</a-button>
      </slot>
      <a-pagination
        v-if="pager"
        :pageSizeOptions="pagination.pageSizeOptions"
        :total="pagination.total"
        v-model="pagination.current"
        :showTotal="pagination.showTotal"
        showSizeChanger
        :pageSize.sync="pagination.pageSize"
        @showSizeChange="pagination.showSizeChange"
        @change="pagination.change"
        :size="pagination.mini?'small':''"
        :style="{float: 'right'}"
      >
        <template slot="buildOptionText" slot-scope="props">
          <span v-if="props.value!=='-1'">{{props.value}}条/页</span>
          <span v-if="props.value==='-1'">全部</span>
        </template>
      </a-pagination>
    </div>
  </div>
</template>
<script>
import XEUtils from "xe-utils";
import { marks } from "./commonConst.js";
export default {
  props: {
    pagingLimit: {
       type:Object,
       default() {
         return null
       }
    },
    messenger: {  //messenger值改变重新把filterData传入vxe-table
      type: Boolean,
      default() {
        return false;
      }
    },
    value: {
      type: [Array],
      default() {
        return [];
      }
    },
    columns: {
      type: Array,
      required: true
    },
    resizable: {
      type: Boolean,
      default() {
        return false;
      }
    },
    border: {
      type: Boolean,
      default() {
        return true;
      }
    },
    stripe: {
      type: Boolean,
      default() {
        return true;
      }
    },
    highlightHoverRow: {
      type: Boolean,
      default() {
        return true;
      }
    },
    calculators: {
      type: Array,
      default() {
        return [];
      }
    },
    pager: {
      type: [Boolean, Object],
      default() {
        return false;
      }
    },
    operations: {
      type: [Array],
      default() {
        return []; //delete, valid, invalid, search(filter), title
      }
    },
    searchColumns: {
      type: [Array],
      default() {
        return ["chrName"];
      }
    },
    modelPath: {
      type: [String],
      default() {
        return "";
      }
    },
    checkboxConfig: {
      type: Object,
      default() {
        return {};
      }
    },
    doCellClick: {
      type: Function,
      default() {
        return () => {};
      }
    }
  },
  data() {
    return {
      loadTag:false,//import异步加载完毕tag
      tableData: [],
      filterData: [],
      params: {},
      searchValue: "",
      loading: false,
      tableColumns: [],
      tableCustoms: [],
      pagination: {
        service: false,
        total: 0,
        current: 1,
        pageSize: 100,
        pageSizeOptions: ["100", "300", "500"],
        showTotal: total => `共 ${total} 条记录`,
        showSizeChange: (current, size) => {
          this.pagination.current = 1;
          this.pagination.pageSize = size;
          let reload = !(this.pagination && !this.pagination.service);

          this.loadData({ reload: reload, paginate: true },this.pagingLimit);
        },
        change: (page, size) => {
          this.pagination.current = page;
          let reload = !(this.pagination && !this.pagination.service);
          this.loadData({ reload: reload, paginate: true },this.pagingLimit);
        }
      },
      deleteConfirmVisible: false,
      model: null,
      xid: "chrId",
      xcode: "chrCode",
      xvalid: "isValid",
      delayOptions: null,
      calculateMethods: [],
      calculateColumns: [],
      customListDrop: false,
      faker: false
    };
  },
  methods: {
    loadData(options = {}, params = {}) {
      setTimeout(() => {
        if(!this.loadTag) {
            this.loadData(options,params);
            return
        }
      this.loading = true;
      let xTable = this.$refs.xTable;
      if (!xTable) {
        this.loading = false;
        return;
      }

      let { reload, page, action, paginate } = Object.assign(
        { reload: true, page: -1, action: () => {}, paginate: false },
        options
      );

      if(paginate) {
        params = Object.assign({}, this.params);
      } else {
        this.params = Object.assign({}, params);
      }
      
      //没有数据来源
      if (!this.model) {
        this.delayOptions = options;
        this.loading = false;
        return;
      }

      if (page != -1 && this.pagination) {
        this.pagination.current = page;
      }
      if (this.pagination && this.pagination.service) {
        params.page = {
          pageNum: this.pagination.current,
          pageSize: this.pagination.pageSize
        };
      }

      if (reload) {
        this.model
          .query(params)
          .then(data => {
            // let tableData = (this.pagination && this.pagination.service) ? data.list : data;
            let tableData = data.hasOwnProperty("list") ? data.list : data;
            this.$emit("input", tableData);
            let pageData =
              this.pagination &&
              !this.pagination.service &&
              this.pagination.pageSize != -1
                ? tableData.slice(
                    this.pagination.pageSize * (this.pagination.current - 1),
                    this.pagination.pageSize * this.pagination.current
                  )
                : tableData;
            this.tableData = [...tableData];
            this.filterData = [...tableData];
            xTable
              .loadData(pageData)
              .then(() => {
                action();
                if (this.pagination) {
                  this.pagination.total =
                    this.pagination && this.pagination.service
                      ? data.total
                      : tableData.length;
                }
                xTable.clearScroll();
                if(!this.checkboxConfig.reserve) {
                  xTable.clearSelection();
                  this.$emit('selectedRows', []);
                }
                if (this.calculateMethods.length) {
                  xTable.updateFooter();
                }
                this.loading = false;
              })
              .catch(err => {
                console.error(err);
                this.loading = false;
              });
          })
          .catch(msg => {
            this.$message.error(msg);
            if (this.faker) {
              const size = 30;
              this.fakeData(size).then(data => {
                xTable.loadData(data);
                this.tableData = [...data];
                this.filterData = [...data];
                if (this.pagination) {
                  this.pagination.total = size;
                }

                this.$emit("input", data);
                action();
                this.loading = false;
              });
            } else {
              this.loading = false;
            }
          });
      } else {
        let tableData =
          this.pagination && this.pagination.pageSize != -1
            ? this.filterData.slice(
                this.pagination.pageSize * (this.pagination.current - 1),
                this.pagination.pageSize * this.pagination.current
              )
            : this.filterData;
        xTable
          .loadData(tableData)
          .then(() => {
            action();
            if (this.pagination) {
              this.pagination.total = this.filterData.length;
            }

            xTable.clearScroll();
            if(!this.checkboxConfig.reserve) {
              this.$emit('selectedRows', []);
            }
            if (this.calculateMethods.length) {
              xTable.updateFooter();
            }
            this.loading = false;
          })
          .catch(err => {
            console.error(err);
            this.loading = false;
          });
      }
      },10);
    },
    assignData(data) {
      let xTable = this.$refs.xTable;
      if (!xTable) {
        return;
      }
      let vm = this;
      xTable
        .loadData(data)
        .then(() => {
          if (vm.pagination) {
            this.pagination.total = data.length;
          }
          vm.tableData = [...data];
          vm.filterData = [...data];
          if (vm.calculateMethods.length) {
            xTable.updateFooter();
          }
          vm.$emit("assigned", true);
        })
        .catch(err => {
          console.error(err);
        });
    },
    clearData() {
      let xTable = this.$refs.xTable;
      if (!xTable) {
        return;
      }

      xTable
        .loadData([])
        .then(() => {
          if (this.pagination) {
            this.pagination.total = 0;
          }
          this.tableData = [];
          this.filterData = [];
          if (this.calculateMethods.length) {
            xTable.updateFooter();
          }
        })
        .catch(err => {
          console.error(err);
        });
    },
    updateFooter() {
      if (this.calculateMethods.length) {
        this.$refs.xTable.updateFooter();
      }
    },
    handleSelectChange({ selection, reserves }) {
      if(this.checkboxConfig.reserve) {
        this.$emit('selectedRows', [...reserves, ...selection]);
      } else {
        this.$emit('selectedRows', selection);
      }
    },
    handleRadioChange({ row }) {
      this.$emit("selectedRows", row);
    },
    getPages() {
      return {
        currentPage: this.pagination.current,
        pages: Math.ceil(this.pagination.total / this.pagination.pageSize)
      };
    },
    nextPage() {
      this.pagination.current++;
      this.loadData();
    },
    prevPage() {
      this.pagination.current--;
      this.loadData();
    },
    filter(value, columns) {
      value = value.trim();
      if (value == "") {
        this.filterData = this.tableData;
      } else {
        let keys = new RegExp(value);
        this.filterData = this.tableData.filter(item => {
          let str = "";
          columns.forEach(x => (str += item[x] ? item[x] : ""));
          return keys.test(str);
        });
      }

      this.loadData({ reload: false, page: 1 });
    },
    checkRow(row) {
      this.$refs.xTable.clearSelection();
      this.$refs.xTable.setSelection(row, true);
      this.$emit('selectedRows', this.$refs.xTable.getSelectRecords());
    },
    checkAll(check = true) {
      this.$refs.xTable.setAllSelection(check);
      this.$emit('selectedRows', this.$refs.xTable.getSelectRecords());
    },
    handleDeleteConfirmVisible() {
      if (this.$refs.xTable.getSelectRecords().length) {
        this.deleteConfirmVisible = true;
      } else {
        this.deleteConfirmVisible = false;
        this.$message.error(marks.noSelectDel);
      }
    },
    handleEliminate(row) {
      this.$refs.xTable.remove(row);
      this.updateFooter();
    },
    handleDelete(row = null) {
      this.deleteConfirmVisible = false;
      if (!this.model) {
        return;
      }

      this.model
        .delete(row ? [row] : this.$refs.xTable.getSelectRecords())
        .then(data => {
          this.$emit('selectedRows', []);
          this.$message.success(marks.delSuc);
          this.loadData({ page: 1 });
        })
        .catch(msg => this.$message.error(msg));
    },
    handleValid(valid, row = null) {
      if (!this.model) {
        return;
      }
      if (!row && !this.$refs.xTable.getSelectRecords().length) {
        this.$message.error(
          valid == "1" ? marks.noSelectEn : marks.noSelectDis
        );
        return;
      }

      this.model
        .changeStatus(row ? [row] : this.$refs.xTable.getSelectRecords(), valid)
        .then(data => {
          if (row) {
            row[this.xvalid] = valid;
          } else {
            this.$refs.xTable.getSelectRecords().map(x => {
              x[this.xvalid] = valid;
            });
          }
          this.$refs.xTable.refreshData();
          this.$message.success(marks.saveSuc);
        })
        .catch(msg => this.$message.error(msg));
    },
    handleSearch() {
      this.filter(this.searchValue, this.searchColumns);
    },
    //用来判断 你想用search title 哪一个
    hasOperation(operations) {
      return this.operations.find(o => operations.indexOf(o) != -1);
    },
    footerMethod({ columns, data }) {
      if (!this.calculateMethods.length) {
        return;
      }
      let methods = [];
      if (this.calculateMethods.includes("sum")) {
        methods.push(
          columns.map((column, columnIndex) => {
            if (columnIndex === 0) {
              return "合计";
            }
            if (!this.pager && columnIndex === 1) {
              return "共" + data.length + "行";
            }
            if (this.calculateColumns.includes(column.property)) {
              return XEUtils.commafy(XEUtils.sum(data, column.property), {
                fixed: 2
              });
            }
            return null;
          })
        );
      }
      return methods;
    },
    fakeData(size) {
      return new Promise(resolve => {
        setTimeout(() => {
          let list = [];
          for (let i = 0; i < size; i++) {
            let item = this.model.newObj();
            item.chrName = this.model.name + (i + 1);
            item[this.xid] = this.xid + (i + 1);
            list.push(item);
          }
          resolve(list);
        }, 250);
      });
    },
    refresh() {
      this.$refs.xTable.refreshData();
    },
    //判断你用search和分页或者下面的按钮从而计算高度
    calTableHeightClass() {
      let str = '';
      str += this.hasOperation(['search', 'title', 'custom']) ? 'has' : 'no';
      str += '-top-';
      str += this.hasOperation(['delete']) || this.pager ? 'has' : 'no';
      str += '-bottom';
      return str;
    }
  },
  watch: {
    messenger: {
      handler(newVal, oldVal) {
        this.assignData(this.filterData);
      },
      deep: true
    }
  },
  created() {
    //处理分页参数
    if (typeof this.pager == "object") {
      Object.assign(this.pagination, this.pager);
    } else if (!this.pager) {
      this.pagination = false;
    }

    //对状态列和操作列的默认slot写入
    this.tableColumns = [...this.columns];
    let statusColumn = this.tableColumns.find(x => x.type == "valid");
    if (statusColumn) {
      statusColumn.slots = {
        default: ({ row }) => {
          return [
            <p
              class={
                row.isValid == "1" ? "operation-enabled" : "operation-disabled"
              }
            >
              {row.isValid == "1" ? "启用" : "停用"}
            </p>
          ];
        }
      };
    }
    let operationColumn = this.tableColumns.find(x => x.type == "operation");
    if (operationColumn) {
      operationColumn.slots = {
        default: ({ row }) => {
          let operations = [];
          if (operationColumn.extra) {
            let extras = operationColumn.extra({ row });
            extras.forEach(e => operations.push(e));
          }
          if (operationColumn.operations.includes("valid")) {
            operations.push(
              <a-popconfirm
                placement="topRight"
                arrowPointAtCenter
                title={row.isValid == "1" ? "确定停用吗?" : "确定启用吗?"}
                onConfirm={() =>
                  this.handleValid(row.isValid == "1" ? "0" : "1", row)
                }
              >
                <a-tooltip>
                  <template slot="title">
                    {row.isValid == "1" ? "停用" : "启用"}
                  </template>
                  <span class="operation-icon">
                    <a-icon type="stop" />
                  </span>
                </a-tooltip>
              </a-popconfirm>
            );
          }
          if (operationColumn.operations.includes("delete")) {
            operations.push(
              <a-popconfirm
                placement="topRight"
                arrowPointAtCenter
                title="确定删除吗?"
                onConfirm={() => this.handleDelete(row)}
              >
                <a-tooltip>
                  <template slot="title">删除</template>
                  <span class="operation-icon">
                    <a-icon type="delete" />
                  </span>
                </a-tooltip>
              </a-popconfirm>
            );
          }
          if (operationColumn.operations.includes("eliminate")) {
            operations.push(
              <a-popconfirm
                placement="topRight"
                arrowPointAtCenter
                title="确定删除吗?"
                onConfirm={() => this.handleEliminate(row)}
              >
                <a-tooltip>
                  <template slot="title">删除</template>
                  <span class="operation-icon">
                    <a-icon type="delete" />
                  </span>
                </a-tooltip>
              </a-popconfirm>
            );
          }
          return operations;
        }
      };
    }

    //加载model
    if (this.modelPath) {
      let unwatch = this.$watch("model", () => {
        if (this.delayOptions) {
          this.loadData(this.delayOptions);
        }
        if (unwatch) {
          unwatch();
        }
      });
      let loadModel = () => import("@/views/js/" + this.modelPath + ".js");
      loadModel()
        .then(data => {
   
          
          this.model = data.default;
          this.loadTag=true;
        })
        .catch(err => console.error(err));
    }

    //处理表尾合计
    if (this.calculators.length) {
      this.calculateMethods = this.calculators[0];
      this.calculateColumns = this.calculators[1];
    }
  }
};
</script>
<style lang="less" scoped>
.no-top-has-bottom {
  height: calc(100% - 60px);
}
.has-top-has-bottom {
  height: calc(100% - 100px);
}
.no-top-no-bottom {
  height: calc(100%);
}
.has-top-no-bottom {
  height: calc(100% - 40px);
}
.top-operations {
  height: 50px;
  padding: 8px 0;
  .top-title {
    float: left;
    line-height: 30px;
  }
  .search {
    float: right;
    width: 300px;
  }
  .custom {
    float: right;
  }
}
li.ant-dropdown-menu-item {
  padding: 10px;
}
.operation-disabled {
  color: @au-pub-danger;
}
.operation-enabled {
  color: @au-pub-success;
}
.operation-icon {
  padding: 0 4px;
}
.bottom-operations {
  height: 60px;
  padding: 15px 0;
  .xtable-bottom-opt-btn {
    margin-right: 8px;
  }
}
</style>