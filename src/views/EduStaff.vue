<script lang="ts" setup>
  import { Search, Edit,User, Plus } from "@element-plus/icons-vue";
  import {ref, onMounted, reactive, nextTick} from "vue";
  import { useStaffStore } from '@/store'
  import {StaffAPI, UserAPI} from '@/api'
  import {ElMessageBox, ElMessage, FormInstance, FormRules} from "element-plus";
  import { httpBatch } from "@/utils/http";


  const useStaff = useStaffStore();
  const staffList = ref<Vm.Staff[]>([]);
  const dicList = ref<Vm.dictionary[]>([]);
  const dic_category = reactive<Vm.dictionary[]>([]);
  const formRef = ref<FormInstance | null>(null);
  const tableRef = ref<any>(null);
  const pagination = reactive({
    total: 0,
    currentPage: 1
  });
  const edit = reactive({
    isEdit: false,
    editType: 0,
    model: {
      stf_id: 0,
      stf_name: '',
      stf_category: 0,
      stf_remark: ''
    }
  });
  const searchModel = reactive<HttpPayload.Staff>({
    stf_id: 0,
    stf_category: 0,
    stf_name: '',
    begin: 0,
    pageSize: 5
  })
  const getData = async (fromPage1 = true) => {
    if(fromPage1) pagination.currentPage = 1;
    searchModel.begin = (pagination.currentPage - 1) * searchModel.pageSize;
    await httpBatch([
      () => {
      return new Promise((resolve, reject) => {
        StaffAPI.get(searchModel)
        .then(r => {
          const { total, list } = r
          if(total > 0 && list.length === 0) {
            pagination.currentPage = Math.ceil(total / searchModel.pageSize);
            getData(false);
            return;
          }
          staffList.value = list;
          pagination.total = total;
          resolve(null);
        })
        .catch(() => reject())
      })
      },
      () => {
      return new Promise((resolve, reject) => {
        StaffAPI.getDic()
        .then( r => {
          dicList.value = r;
          resolve(null);
        } )
        .catch(() => reject())
      })
      }
    ])
  }
  onMounted(async () => {
    console.log(staffList)
    try {
      await getData();
      dicList.value.forEach(item => {
        if(item.dic_group_key === 'staff_category')
          dic_category.push(item);
      })
    }catch (e) {}
  });
  // ??????
  const setDimission  = async (staff: HttpPayload.Staff) => {
      try {
        await ElMessageBox.confirm(
            `????????????${staff.stf_name}?????????????????????????????????`,
            'Warning',
            { type: 'warning', confirmButtonText: '??????', cancelButtonText: '??????' }
        );
        await httpBatch([() => StaffAPI.setDimission(staff.stf_id), () => getData()], true)
        ElMessage({
          type: 'success',
          message: `"${staff.stf_name}"????????????????????????????????????`
        })
      } catch (e) {}
  };
  // ??????
  const setReinstate = async (staff: HttpPayload.Staff) => {
    try {
      await ElMessageBox.confirm(
          `????????????${staff.stf_name}?????????????????????????????????`,
          'Warning',
          { type: 'warning', confirmButtonText: '??????', cancelButtonText: '??????' }
      );
      await httpBatch([ () => StaffAPI.setReinstate(staff.stf_id), () => getData() ], true)
      ElMessage({
        type: 'success',
        message: `"${staff.stf_name}"????????????????????????????????????`
      })
    }catch(e) {}
  };
  // ??????
  const beginAdd = async () => {
    // edit.editType = 1;
    edit.model.stf_id = 0
    edit.model.stf_name = '';
    edit.model.stf_category = 0;
    edit.model.stf_remark = '';
    await nextTick(() => edit.isEdit = true );
  };
  const beginUpdate = async (staff: HttpPayload.StaffEdit) => {
    // edit.editType = 2;
    edit.model.stf_id = staff.stf_id;
    edit.model.stf_name = staff.stf_name;
    edit.model.stf_category = staff.stf_category;
    edit.model.stf_remark = staff.stf_remark;
    await nextTick(() => edit.isEdit = true );
  };
  const save = async () => {
      if(edit.model.stf_id === 0) {
        await formRef.value?.validateField();
        await httpBatch([ () => StaffAPI.add(edit.model), () => getData() ], true);
        // await StaffAPI.add(edit.model);
        // staffList.value.push({ ...edit.model, stf_invalid: 1, })
        ElMessage({
          message: `??????"${edit.model.stf_name}"???????????????`,
          type: 'success'
        });
        edit.isEdit = false;
        // ???????????????????????????
        await nextTick(() => {
          let i = staffList.value.findIndex(item => item.stf_name === edit.model.stf_name);
          tableRef.value.setCurrentRow(staffList.value[i]);
        });
      } else {
        await httpBatch([ () => StaffAPI.update(edit.model), () => getData(false) ], true);
        ElMessage({
          message: `??????"${edit.model.stf_name}"???????????????`,
          type: 'success'
        });
        edit.isEdit = false;
      }
  };
  const validatePass = (rule: any, value: any, callback: any) => {
    if (value === 0) {
      callback(new Error('* ????????????????????????'))
    } else {
      if (edit.model.stf_category !== null) {
        if (!formRef.value) return
        formRef.value.validateField('checkPass', () => null)
      }
      callback()
    }
  };

  const rules: FormRules = {
    stf_name: [
      { required: true, message: '* ???????????????????????????', trigger: 'blur' },
      { min: 2, max: 20, message: '* ??????????????????2-20?????????', trigger: 'blur' }
    ],
    stf_category: [{ required: true, validator: validatePass, trigger: 'blur' }]
  };
  const getIndex = (rowIndex: number) => {
    return (pagination.currentPage - 1) * searchModel.pageSize + rowIndex + 1;
  }
</script>

<template>
  <el-container>
    <el-header>
      <el-form inline>
        <el-form-item label="????????????:">
          <el-input :prefix-icon="Search"
                    v-model="searchModel.stf_name"
                    placeholder="?????????????????????"
                    @change="getData()"></el-input>
        </el-form-item>
        <el-form-item label="????????????:">
          <el-select v-model="searchModel.stf_category" @change="getData()">
            <el-option :value="0" label="- ????????? -"></el-option>
            <el-option :value="item.dic_id" :label="item.dic_name" v-for="item in dic_category" :key="item.dic_id"></el-option>
          </el-select>
        </el-form-item>
        <el-form-item style="color: #ffffff">
          <el-button type="primary" :icon="Plus" style="width: 200px" @click="beginAdd">??????</el-button>
        </el-form-item>
      </el-form>
    </el-header>
    <el-main>
      <el-table :data="staffList"
                border
                stripe
                highlight-current-row
                ref="tableRef">
        <el-table-column label="#" :index="getIndex" prop="#" type="index" align="center"></el-table-column>
        <el-table-column label="??????" align="center" prop="stf_name"></el-table-column>
        <el-table-column label="????????????" align="center">
          <template #default="{row}">
            <span>{{ dic_category.find(item => item.dic_id === row.stf_category)?.dic_name }}</span>
          </template>
        </el-table-column>
        <el-table-column label="??????" align="center">
          <template #default="{row}">
            <span v-if="row.stf_remark !== ''">{{row.stf_remark}}</span>
            <span v-else style="color: #CDD0D6">??????????????????</span>
          </template>
        </el-table-column>
        <el-table-column label="????????????" align="center">
          <template #default="{row}">
            <span v-if="row.stf_invalid === 1" style="color: #409EFF">??????</span>
            <span v-else style="color: #F56C6C">??????</span>
          </template>
        </el-table-column>
        <el-table-column label="??????" align="center">
          <template #default="{row}">
            <el-button type="primary" :icon="Edit" @click="beginUpdate(row)">??????</el-button>
            <el-button type="success" :icon="User" v-if="row.stf_invalid === 0" @click="setReinstate(row)">??????</el-button>
            <el-button type="danger" :icon="User" v-else @click="setDimission(row)">??????</el-button>
          </template>
        </el-table-column>
      </el-table>
<!--   ??????/????????????   -->
      <el-dialog v-model="edit.isEdit"
                 width="500px"
                 center
                 draggable
                 :close-on-click-modal="false"
                 :close-on-press-escape="false"
                 :show-close="false" >
        <template #header>
          <span>{{edit.model.stf_id === 0? "????????????" : "??????????????????"}}</span>
        </template>
        <el-form ref="formRef" :model="edit.model" status-icon :rules="rules" label-width="120px">
          <el-form-item label="???????????????" prop="stf_name">
            <el-input placeholder="???????????????"
                      v-model="edit.model.stf_name"></el-input>
          </el-form-item>
          <el-form-item label="???????????????" prop="stf_category">
            <el-select v-model="edit.model.stf_category">
              <el-option label="- ????????? -" :value="0"></el-option>
              <el-option v-for="item in dic_category"
                         :key="item.dic_id"
                         :label="item.dic_name"
                         :value="item.dic_id"
              ></el-option>
            </el-select>
          </el-form-item>
          <el-form-item label="???????????????">
            <el-input type="textarea" placeholder="?????????????????????" v-model="edit.model.stf_remark"></el-input>
          </el-form-item>
        </el-form>
        <template #footer>
      <span class="dialog-footer">
        <el-button @click="edit.isEdit = false">??????</el-button>
        <el-button type="primary" @click="save">??????</el-button>
      </span>
        </template>
      </el-dialog>
    </el-main>
    <el-footer>
      <el-pagination
          background
          layout="pre,pager,next,jumper,->,sizes, total"
          :page-sizes="[5,9,14,18,22]"
          :total="pagination.total"
          v-model:page-size="searchModel.pageSize"
          v-model:current-page="pagination.currentPage"
          @size-change="getData()"
          @current-change="getData(false)"
      />
    </el-footer>
  </el-container>
</template>

<style scoped lang="stylus">
.el-container
  display: flex
  height 100%
  .el-header
    flex-shrink: 0
  .el-main
    flex-grow: 1
  el-footer
    flex-shrink: 0
</style>
