<template xmlns:el-col="http://www.w3.org/1999/html">
  <el-col v-loading="loading" class="tree-main">
    <el-row v-if="showExtent" class="tree-head">
      <span style="float: left;padding-left: 10px">{{ dataInfo.head }}</span>
      <span v-for="auth in defaultAuthDetails" :key="auth.privilegeName" class="auth-span">
        {{ auth.privilegeName }}
      </span>
    </el-row>
    <el-row style="margin-top: 5px">
      <el-tree
        :props="defaultProps"
        :load="loadNodes"
        :data="treeData"
        :node-key="defaultProps.id"
        :highlight-current="highlightCurrent"
        :default-expanded-keys="expandedKey"
        lazy
        @node-click="nodeClick"
      >
        <span slot-scope="{ node, data }" class="custom-tree-node">
          <span>
            <span style="margin-left: 6px" v-html="data.name" />
          </span>
          <span v-if="showExtent" @click.stop>
            <div v-if="authDetails[data.id]">
              <span v-for="auth in authDetails[data.id]" :key="auth.privilegeType" class="auth-span">
                <!-- 1-{{ auth.privilegeType }}-{{ auth.privilegeValue }}-->
                <a href="javascript:;" @click="clickAuth(data.id,auth)">
                  <svg-icon style="width: 25px;height: 25px" :icon-class="auth.privilegeValue===1?'lock_open':'lock_closed'" />
                </a>
              </span>
            </div>
            <div v-else>
              <span v-for="auth in defaultAuthDetails" :key="auth.privilegeType" class="auth-span">
                <!--2-{{ auth.privilegeType }}-{{ auth.privilegeValue }}-->
                <a href="javascript:;" @click="clickAuth(data.id,auth)">
                  <svg-icon style="width: 25px;height: 25px" :icon-class="auth.privilegeValue===1?'lock_open':'lock_closed'" />
                </a>
              </span>
            </div></span>
        </span>
      </el-tree>
    </el-row>
  </el-col>
</template>

<script>
// import { authChange, authDetails, authDetailsModel, authModel } from '@/api/system/sysAuth'
// import { execute } from '@/de-base/api/de-api'
export default {
  name: 'LazyTree',
  components: { },
  props: {
    filterText: {
      type: String,
      required: false,
      default: ''
    },
    authCondition: {
      type: Object,
      required: false
    },
    dataInfo: {
      type: Object,
      required: true
    },
    activeName: {
      type: String,
      required: true
    },
    attachActiveName: String,
    defaultProps: {
      type: Object,
      required: false,
      default: function () {
        return {
          children: 'children',
          label: 'name',
          id: 'id',
          parentId: 'pid',
          isLeaf: 'leaf'
        }
      }
    },
    showExtent: Boolean,
    highlightCurrent: Boolean
  },
  data () {
    return {
      loading: false,
      treeData: [],
      changeIndex: 0,
      timeMachine: null,
      expandedKey: [], // ???????????? ?????????????????????????????????
      defaultCondition: { // pid ???0????????? ???????????????????????????
        pid: '0'
      },
      authDetails: {},
      defaultAuthDetails: [],
      searchStatus: false, // ??????????????????????????? ??????????????? ???????????????????????????
      // ???????????????????????????ID ????????????????????????authTarget??????????????????????????????
      loadedNodeIds: new Set()
    }
  },
  computed: {
  },
  watch: {
    filterText (val) {
      this.expandedKey = []
      if (val && val.length > 0) {
        this.searchStatus = true
      }
      // ??????????????? activeName ????????? ???????????????
      if (this.dataInfo.authType === this.activeName) {
        this.destroyTimeMachine()
        this.changeIndex++
        this.filterNode(this.changeIndex)
      }
    },
    authCondition: {
      handler (newVal, oldVla) {
        this.loadAuth()
      },
      deep: true
    },
    activeName: {
      handler (newVal, oldVla) {
        this.loadAuth()
      },
      deep: true
    },
    attachActiveName: {
      handler (newVal, oldVla) {
        this.authDetails = {}
      },
      deep: true
    }
  },
  created () {
    // ?????????????????????
    if (this.showExtent) {
      this.executeAxios('/plugin/auth/authDetailsModel/' + this.dataInfo.authType, 'get', {}, res => {
        this.defaultAuthDetails = res.data
      })
      //   authDetailsModel(this.dataInfo.authType).then(res => {
      //     this.defaultAuthDetails = res.data
      //   })
      this.loadAuth()
    }
  },
  methods: {
    executeAxios (url, type, data, callBack) {
      const param = {
        url: url,
        type: type,
        data: data,
        callBack: callBack
      }
      this.$emit('execute-axios', param)
      // if (process.env.NODE_ENV === 'development') {
      //   execute(param).then(res => {
      //     if (param.callBack) {
      //       param.callBack(res)
      //     }
      //   }).catch(e => {
      //     if (param.error) {
      //       param.error(e)
      //     }
      //   })
      // }
    },
    loadAuth () {
      if (this.authCondition && this.showExtent) {
        let authQueryCondition = {}
        if (this.dataInfo.direction === 'source') {
          // ????????????????????? ????????????authTarget ??????????????? authSource
          authQueryCondition = {
            authTarget: this.authCondition.id,
            authTargetType: this.authCondition.type,
            authSourceType: this.dataInfo.authType
          }
        } else {
          authQueryCondition = {
            authSource: this.authCondition.id,
            authSourceType: this.authCondition.type
          }
        }
        this.executeAxios('/plugin/auth/authDetails', 'post', authQueryCondition, res => {
          this.authDetails = res.data
        })
        // authDetails(authQueryCondition).then(res => {
        //   this.authDetails = res.data
        // })
      }
    },
    loadNodes (node, resolve) {
      if (!this.searchStatus) {
        if (node.level === 0) {
          const queryCondition = {
            modelType: this.dataInfo.authType,
            ...this.defaultCondition
          }
          this.executeAxios('/plugin/auth/authModels', 'post', queryCondition, res => {
            const data = res.data
            resolve(data)
          })
        //   authModel(queryCondition).then(res => {
        //     const data = res.data
        //     resolve(data)
        //   })
        } else {
          const queryCondition = {
            modelType: this.dataInfo.authType
          }
          queryCondition[this.defaultProps.parentId] = node.data[this.defaultProps.id]
          this.executeAxios('/plugin/auth/authModels', 'post', queryCondition, res => {
            const data = res.data
            resolve(data)
          })
        //   authModel(queryCondition).then(res => {
        //     const data = res.data
        //     resolve(data)
        //   })
        }
      } else {
        resolve(node.data.children)
      }
    },
    filterNode (index) {
      this.timeMachine = setTimeout(() => {
        if (index === this.changeIndex) {
          const queryCondition = {
            withExtend: 'parent',
            modelType: this.dataInfo.authType
          }
          queryCondition[this.defaultProps.label] = this.filterText
          this.executeAxios('/plugin/auth/authModels', 'post', queryCondition, res => {
            // ????????????
            this.highlights(res.data)
            this.treeData = this.buildTree(res.data)
            // ??????searchStatus ?????? ??????????????????????????????
            this.$nextTick(() => (this.searchStatus = false))
          })
        //   authModel(queryCondition).then(res => {
        //     // ????????????
        //     this.highlights(res.data)
        //     this.treeData = this.buildTree(res.data)
        //     // ??????searchStatus ?????? ??????????????????????????????
        //     this.$nextTick(() => (this.searchStatus = false))
        //   })
        }
        this.destroyTimeMachine()
      }, 1500)
    },
    nodeClick (data, node) {
      this.$emit('nodeClick', { id: data.id, type: this.dataInfo.authType })
    },
    destroyTimeMachine () {
      this.timeMachine && clearTimeout(this.timeMachine)
      this.timeMachine = null
    },
    buildTree (arrs) {
      const idMapping = arrs.reduce((acc, el, i) => {
        acc[el[this.defaultProps.id]] = i
        return acc
      }, {})
      const roots = []
      arrs.forEach(el => {
        // ??????????????? ###
        if (el[this.defaultProps.parentId] === null || el[this.defaultProps.parentId] === 0 || el[this.defaultProps.parentId] === '0') {
          roots.push(el)
          return
        }
        // ???????????????????????????
        const parentEl = arrs[idMapping[el[this.defaultProps.parentId]]]
        // ????????????????????????????????????`children`?????????
        parentEl.children = [...(parentEl.children || []), el]

        // ?????????????????? ???????????????????????????????????????
        if (parentEl.children.length > 0) {
          this.expandedKey.push(parentEl[this.defaultProps.id])
        }
      })
      return roots
    },
    // ????????????
    clickAuth (dataId, auth) {
      let authChangeCondition = {}
      if (this.dataInfo.direction === 'source') { // ?????????????????????
        authChangeCondition = {
          authSource: dataId,
          authSourceType: this.dataInfo.authType,
          authTarget: this.authCondition.id,
          authTargetType: this.authCondition.type,
          authDetail: auth

        }
      } else {
        authChangeCondition = {
          authTarget: dataId,
          authTargetType: this.dataInfo.authType,
          authSource: this.authCondition.id,
          authSourceType: this.authCondition.type,
          authDetail: auth
        }
      }
      this.loading = true
      this.executeAxios('/plugin/auth/authChange', 'post', authChangeCondition, res => {
        // ??????????????????
        this.loadAuth()
        this.loading = false
      })
    },
    // ????????????????????????
    highlights (data) {
      if (data && this.filterText && this.filterText.length > 0) {
        const replaceReg = new RegExp(this.filterText, 'g')// ?????????????????????
        const replaceString = '<span style="color: #faaa39">' + this.filterText + '</span>' // ????????????v-html???
        data.forEach(item => {
          item.name = item.name.replace(replaceReg, replaceString) // ????????????
        })
      }
    }

  }
}
</script>

<style scoped>
  .custom-tree-node {
    flex: 1;
    display: flex;
    align-items: center;
    justify-content: space-between;
    font-size: 14px;
    padding-left: 8px;
  }
  .tree-main{
    overflow-y: auto;
  }
  .blackTheme .tree-main {
    border-color: var(--TableBorderColor) !important;
}
  /* .tree-head{
    height: 30px;
    line-height: 30px;
    border-bottom: 1px solid #e6e6e6;
    background-color: #f7f8fa;
    font-size: 12px;
    color: #3d4d66 ;
  } */

  .tree-head{
    height: 30px;
    line-height: 30px;
    border-bottom: 1px solid var(--TableBorderColor, #e6e6e6);
    background-color: var(--SiderBG, #f7f8fa);
    font-size: 12px;
    color: var(--TableColor, #3d4d66) ;
  }

  .auth-span{
    float: right;
    width:50px;
    margin-right: 30px
  }
  .highlights-text {
    color: #faaa39 !important;
  }

</style>
