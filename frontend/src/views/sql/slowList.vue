<style scoped>
  .time{
    font-size: 14px;
  }
  .content{
    font-weight: bold;
    padding-left: 5px;
  }
</style>

<template>
    <div>
        <div>
          <Card>
            <Row>
              <Col span="4">
                <Input icon="search" v-model="getParams.search" placeholder="搜索" @on-click="handleGetSqlList" @on-enter="handleGetSqlList" />
              </Col>
              <Col span="6">
                <DatePicker type="daterange" :options="dateOption" @on-change="dateChange" @on-clear="dateClear" placement="bottom-end" placeholder="选择日期范围" style="width: 200px; float:right"></DatePicker>
              </Col>
            </Row>  
            </br>
            <Row>
              <Table :columns="columnsSqlList" :data="sqlList" size="small" @on-sort-change='changeSort'></Table>
              </br>
              <Page :total=total show-sizer :current=getParams.page @on-change="pageChange" @on-page-size-change="sizeChange"></Page>
            </Row>
          </Card>
          <copyright> </copyright>
        </div>
        <Spin size="large" fix v-if="spinShow"></Spin>
        <Modal
            v-model="sqlContentModal"
            width="650"
            :title="sqlContentTitle"
            @on-ok="ok"
            @on-cancel="cancel">
            <div>
              <Scroll height=450>
                 <Form ref="createUserForm"  :label-width="100">
                  <Row>
                    <Col span="32">
                        <FormItem label="SQL语句：">
                          <div>{{sqlContent}}</div>
                        </FormItem>
                    </Col>
                    </Row>
                    <Row>
                    <Col span="32">
                        <FormItem label="问题描述：">
                          <i-input  v-model="formdata.content" type="textarea" :rows="4" placeholder="请输入问题描述..." ></i-input>
                        </FormItem>
                    </Col>
                    </Row>
                    <Row>
                      <Col span="24">      
                        <FormItem label="指定执行人：">          
                          <Transfer
                            :data="userList"
                            :target-keys="targetKeysupdate"
                            filterable
                            :filter-method="filterMethod"
                            @on-change="handleChangeupdate"
                            :list-style="listStyle"
                            :titles="transferTitles"
                            >
                          </Transfer>
                        </FormItem>
                      </Col>
                    </Row>
                  </Form>
              </Scroll>
            </div>
        </Modal>  

        <Modal
            v-model="stepsModal"
            width="400"
            :title="stepsModalTitle"
            @on-cancel="cancel">
            <div>
              <Scroll height=450>
                <Timeline>
                  <TimelineItem v-for="(item, index) in steps" :value="item.value" :key="index" :color="getColor(item.status)">
                    <p class="content">{{ item.username }}  </p>
                    <p class="time">{{ item.updatetime | formatTime }} <Tag :color="stepStatusMap[item.status]['color']" style="margin-left:10px">{{ stepStatusMap[item.status]['desc'] }}</Tag></p>
                  </TimelineItem>
                </Timeline>
              </Scroll>
            </div>
        </Modal>  

     </div>
</template>
<script>
  import {GetSqlList, GetUsers} from '@/api/sql/slowquery'
  import {Putdata} from '@/api/sql/slowmyorder'
  import {getSqlContent} from '@/utils/sql/inception'
  import {addDate} from '@/utils/base/date'
  import {handleBadgeData} from '@/utils/sql/inception'
  import baseData from '@/utils/sql/data'
  import copyright from '../my-components/public/copyright'
  import {Button, Table, Page, Modal, Message, Icon, Tag, DropdownMenu, DropdownItem, Dropdown, Tooltip, Poptip, Badge} from 'iview';

  export default {
    components: {Button, Table, Page, Modal, Message, Icon, Tag, DropdownMenu, DropdownItem, Dropdown, Tooltip, Poptip, Badge, copyright},
    filters:{
      formatTime:function(value){
        if(value != '') {
          return value.slice(0,19).replace('T',' ')
        }
      },
    },
    computed:{

    },
    data () {
      return {
        spinShow:false,
        steps:[],
        formdata: {
          MySQLTotalExecutionCounts: '',
          MaxLockTime: '',
          SQLText: '',
          ReturnTotalRowCounts: '',
          ParseTotalRowCounts: '',
          MySQLTotalExecutionTimes: '',
          DBName: '',
          content: '',
          distributor:'',
          executor:'',
          id:''
        },
        stepsModal:false,
        stepsModalTitle:'',
        stepStatusMap:{
          '-1':{color:'gray', desc:'终止', stepStatus:'wait'},
          0:{color:'gray', desc:'待处理'},
          1:{color:'green', desc:'通过'},
          2:{color:'red', desc:'驳回'},
          3:{color:'red', desc:'放弃'}
        },
        total:1,
        getParams:{
          sorttype:'asc',
          sortfield:'',
          page:1,
          pagesize:10,
          search:'',
          daterange:''
        },
        targetKeysCreate:[],
        targetKeysupdate:[],
        listStyle: {
          width: '220px',
          height: '220px'
        },
        transferTitles: ['执行人列表', '被指定的执行人'],
        userList:[],
        getParams:{
          page:1,
          pagesize:10,
          search:'',
        },
        identity:0,
        sqlContentTitle:'',
        sqlContent:'',
        sqlContentModal: false,
        sqlList:[],
        dateOption:baseData.dateOption,
        columnsSqlList: [
          {
            title: 'ID',
            width: 60,
            key:'id',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              // return h('router-link', {props:{to:'/slowquerysql/'+params.row.SQLIdStr}}, params.row.id)
              return h('div', [
                h('span', {}, params.row.id),
              ])
            }
          },

          {
            title: '解析总行数',
            width: 70,
            key:'ParseMaxRowCount',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.ParseMaxRowCount),
              ])
            }
          },

          {
            title: '执行总次数',
            width: 70,
            key:'MySQLTotalExecutionCounts',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.MySQLTotalExecutionCounts),
              ])
            }
          },

          {
            title: 'SQLId',
            width: 100,
            key:'SQLId',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.SQLIdStr),
              ])
            }
          },

          {
            title: 'SQL查询语句',
            width: 300,
            render: (h, params) => {
              // return h('router-link', {props:{to:'/slowquerysql/'+params.row.SQLIdStr}}, params.row.SQLText)
              return h('div', [
                h('span', {}, params.row.SQLText),
              ])
            }
              // return h('div', [
              //   h('span', {}, params.row.SQLText),
              //   // h('Button', {
              //   //   props: {
              //   //     size: 'small',
              //   //   },
              //   //   style: {float:'right'},
              //   //   on: {
              //   //     click: () => {
              //   //       this.sqlContent = getSqlContent(params.row.SQLText)
              //   //       this.sqlContentModal = true
              //   //       this.sqlContentTitle = 'SQL语句' + '（工单ID: ' + params.row.id + '）'
              //   //     }
              //   //   }
              //   // }, '语句')
              // ])
            //}
          },

           {
            title: '锁定最大时长，单位：秒',
            width: 90,
            key:'MaxLockTime',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.MaxLockTime),
              ])
            }
          },

           {
            title: '返回总行数',
            width: 70,
            key:'ReturnTotalRowCounts',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.ReturnTotalRowCounts),
              ])
            }
          },

           {
            title: '锁定总时长，单位：秒',
            width: 90,
            key:'TotalLockTimes',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.TotalLockTimes),
              ])
            }
          },

          {
            title: '执行最大时长，单位：秒',
            width: 90,
            key:'MaxExecutionTime',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.MaxExecutionTime),
              ])
            }
          },

          {
            title: '数据生成日期',
            width: 90,
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.CreateTime),
              ])
            }
          },

           {
            title: '慢查询Id',
            width: 100,
            key:'SlowLogId',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.SlowLogId),
              ])
            }
          },

          {
            title: '解析总行数',
            width: 70,
            key:'ParseTotalRowCounts',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.ParseTotalRowCounts),
              ])
            }
          },

          {
            title: '执行总时长，单位：秒',
            width: 90,
            key:'MySQLTotalExecutionTimes',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.MySQLTotalExecutionTimes),
              ])
            }
          },

          {
            title: '数据库名称',
            width: 90,
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.DBName),
              ])
            }
          },

          {
            title: '返回最大行数',
            width: 70,
            key:'ReturnMaxRowCount',
            sortable: true,
            sortMethod:function(a,b,type){
            },
            render: (h, params) => {
              return h('div', [
                h('span', {}, params.row.ReturnMaxRowCount),
              ])
            }
          },

          // {
          //   title: '发起人',
          //   key: 'commiter'
          // },

          // {
          //   title: '环境',
          //   width: 120,
          //   render: (h, params) => {
          //     const envMap = {
          //       'test':{color:'gray', desc:'测试'},
          //       'prd':{color:'orange', desc:'生产'}
          //     }
          //     const env = params.row.env
          //     return h(Tag, {props:{type:'dot', color:envMap[env]['color']}}, envMap[env]['desc'])
          //   }
          // },

          // {
          //   title: '目标库',
          //   key: 'db_name',
          // },

          

          // {
          //   title: '流程',
          //   width: 100,
          //   render: (h, params) => {
          //     const statusMap = {
          //       1:'success',
          //       2:'warning'
          //     }
          //     const steps = params.row.steps
          //     let badgeData = handleBadgeData(steps)
          //     if (steps.length > 0) {
          //       var subElement = 
          //       [
          //         h(Button, {
          //             props: {
          //               size: 'small',
          //               ghost: true
          //             },
          //             style: {},
          //             on: {
          //               click: () => {
          //                 this.stepsModalTitle = '工单流程' + '（ID: ' + params.row.id + '）'
          //                 this.steps = steps
          //                 this.stepsModal = true
          //               }
          //             }
          //           }, '流程'),
          //         h(Badge, {props:{count:badgeData.count, type:statusMap[badgeData.badgeStatus] }},[])
          //       ]

          //     } else {
          //       subElement = []
          //     }
          //     return h('div', {}, subElement)
          //   }
          // },

          // {
          //   title: '备注',
          //   width: 100,
          //   render: (h, params) => {
          //     let remark = params.row.remark
          //     if (remark.length >= 6){
          //       var abbreviatedRemark = params.row.remark.slice(0,4) + '...'
          //     } else {
          //       var abbreviatedRemark = remark
          //     }
          //     return h(Tooltip,{props:{placement: "top", content: remark}}, 
          //     [
          //       h('div', {props:{slot:'content'}}, [h('div',{}, abbreviatedRemark)])
          //     ])

          //   }
          // },

          // {
          //   title: '工单状态',
          //   width: 120,
          //   render: (h, params) => {
          //     let status = params.row.status
          //     if (status == -4) {
          //       return h('div', [h(Tag,{props:{color:'red'}}, '回滚失败')])
          //     } else if (status == -3) {
          //       return h('div', [h(Tag,{props:{}}, '已回滚')])
          //     } else if (status == -2) {
          //       return h('div', [h(Tag,{props:{}}, '已暂停')])
          //     } else if (status == -1) {
          //       return h('div', [h(Tag,{props:{color:'blue'}}, '待执行')])
          //     } else if (status == 0) {
          //       return h('div', [h(Tag,{props:{color:'green'}}, '成功')])
          //     } else if (status == 1) {
          //       return h('div', [h(Tag,{props:{color:'yellow'}}, '已放弃')])
          //     } else if (status == 2) {
          //       return h('div', [h(Tag,{props:{color:'red'}}, '任务失败')])
          //     } 
          //   }
          // },

          // {
          //   title: '核准人',
          //   key: 'treater'
          // },
          
          {
            title: '操作',
            width: 150,
            align: 'center',
            render: (h, params) => {
              const id = params.row.id
              const status = params.row.slow_status
              const rollbackable = params.row.rollback_able
              const type = params.row.type
              const handleable = params.row.handleable
              const is_manual_review = params.row.is_manual_review
              let popcss = {
                width:170,
                place:'top',
              }
              var ddItem = [ 
                  h('div' , {}, [h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.width, transfer:true, title:'执行 此转化为工单操作？'}, on:{'on-ok': () => {
                    this.sqlContent = params.row.SQLText
                    this.formdata.MySQLTotalExecutionCounts = params.row.MySQLTotalExecutionCounts
                    this.formdata.MaxLockTime = params.row.MaxLockTime
                    this.formdata.SQLText = params.row.SQLText
                    this.formdata.ReturnTotalRowCounts = params.row.ReturnTotalRowCounts
                    this.formdata.ParseTotalRowCounts = params.row.ParseTotalRowCounts
                    this.formdata.MySQLTotalExecutionTimes = params.row.MySQLTotalExecutionTimes
                    this.formdata.DBName = params.row.DBName
                    this.formdata.id = params.row.id
                    this.sqlContentModal = true
                    this.sqlContentTitle = '慢查询SQL语句转工单'
                  } } }, [h(DropdownItem, {}, '转化为工单')] ) ]) ,
              ]


              // if (status == -1) {
              //   var ddItem = [ 
              //     h('div' , {}, [h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.width, transfer:true, title:'执行 此工单(' + id + ') ？'}, on:{'on-ok': () => {this.handleAction('execute', params)} } }, [h(DropdownItem, {}, '执行')] ) ]) , 
              //     h('div' , {}, [h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.place, transfer:true, title:'放弃 此工单(' + id + ') ？'}, on:{'on-ok': () => {this.handleAction('reject', params)} } }, [h(DropdownItem, {}, '放弃')] ) ]),
              //     h('div' , {style:{display: is_manual_review == false || handleable == true  || status == -2 ? 'none' : 'display'}}, [h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.place, transfer:true, title:'审批通过 此工单(' + id + ') ？'}, on:{'on-ok': () => {this.handleAction('approve', params)} } }, [h(DropdownItem, {}, '审批通过')] ) ]),
              //     h('div' , {style:{display: is_manual_review == false || handleable == true  || status == -2 ? 'none' : 'display'}}, [h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.place, transfer:true, title:'审批驳回 此工单(' + id + ') ？'}, on:{'on-ok': () => {this.handleAction('disapprove', params)} } }, [h(DropdownItem, {}, '审批驳回')] ) ]),
              //   ]
              // } else if (status == 0){
              //   var ddItem = [ h(Poptip,{props:{confirm:true, placement:popcss.place, width:popcss.width, transfer:true, title:'回滚 此工单(' + id + ') ？'}, on:{'on-ok': () => {this.handleAction('rollback', params)} } }, [h(DropdownItem, {}, '回滚')] ) ]
              // } else {
              //   //var ddItem = []
              // }
              if (status != 0){
                  return h('div', [
                        h('span',[h(Tag,{props:{color:'green'}}, '已转工单')]),
                  ])
              }else{
                  return h('div', {style:{display: this.identity == 1 ? 'display' : 'none'}}, [
                    h(Dropdown,
                    {
                      style: {marginLeft: '20px'},
                    },
                    [
                      h(Button, {props:{type: 'primary', size: 'small'}}, [h('span', {style:{marginRight: '1px'}}, '操作'), h(Icon, {props:{type: 'arrow-down-b'}})], ),
                      h(DropdownMenu, 
                        {
                          slot: 'list',
                        }, 
                        ddItem               
                      )
                    ], ),
              ])
              }
              
            }
          },
        ],

      }
    },

    created () {
      this.handleGetSqlList()
      this.handleGetUserList()
    },

    methods: {
      getColor(status){
        return this.stepStatusMap[status]['color']
      },
      ok(){
        if (this.targetKeysupdate.length <1){
          this.notice('转化工单失败','没有指定执行人');
          return false;
        }
        this.formdata.executor=this.targetKeysupdate[0];
        Putdata(this.formdata)
        .then(response => {
            console.log(response)
            let httpstatus = response.status
            if (httpstatus == 201) {
              let title = '服务器提示'
              let msg = '慢查询转工单成功'
              this.notice(title, msg)
            }
            var ishttps = 'https:' == document.location.protocol ? true: false;
            var url = window.location.host;
            if(ishttps){
                url = 'https://' + url;
            }else{
                url = 'http://' + url;
            }
            window.location.href= url + '/slowquery/sloworderList';
          })
      },
      handleGetUserList () {
        GetUsers(this.getParams)
        .then(response => {
          this.userFormat(response.data)          
        })
      },
      handleChangeupdate (newTargetKeys) {
        if (this.targetKeysupdate.length >1 || newTargetKeys.length>1){
          this.notice('指定执行人出错','最多只能指定一个人');
          return false;
        }else{
          this.targetKeysupdate = newTargetKeys;
        }
      },
      filterMethod (data, query) {
        return data.label.indexOf(query) > -1;
      },
      userFormat (userlist) {
        let users = []
        userlist.map( (item) => {
          users.push({
            key:item.username,
            label:item.email,
            description:item.id,
            disabled:false
          })
        })
        this.userList = users
      },
      notice (title, msg) {
        this.$Notice.error({
          title: title,
          duration: 6,
          desc: msg
        });
      },
      changeSort(obj){
        this.getParams.sorttype=obj.order
        this.getParams.sortfield=obj.key
        this.handleGetSqlList()
      },
      alertSuccess (title, paramId, execute_time, affected_rows) {
        this.$Notice.success({
          title: title,
          render: h => {
            let id = h('p', {}, 'ID：' + paramId) 
            let time = execute_time ? h('p', {}, '耗时（秒）：' + execute_time) : ''
            let rows = affected_rows ? h('p', {}, '影响的行数：' + affected_rows) : ''
            let subtags = [id, time, rows]
            return h('div', subtags)
          }
        });
      },

      alertWarning (title, paramId) {
        this.$Notice.warning({
          title: title,
          duration: 0,
          render: h => {
            let id = h('p', {}, 'ID：' + paramId) 
            let desc = h('p', {}, '具体查看工单详情[inception结果]') 
            let subTags = [id, desc]
            return h('div', subTags)
          }
        });
      },

      getDatetime () {
        let date = this.userInfo.date_joined || ''
        return date.slice(0,19).replace('T',' ')
      },

      handleGetSqlList () {
        this.spinShow = true
        GetSqlList(this.getParams)
        .then(response => {
          this.spinShow = false
          this.sqlList = response.data.data
          this.total = response.data.counts
          this.identity = response.data.identity
        })
      },

      handleAction (action, params) {
        let id = params.row.id
        SqlAction(id, action)
        .then(response => {
          const status = response.data.status
          const data = response.data.data
          if (status == 0) {
            if (action == 'execute') {
              this.alertSuccess('执行成功', id, data.execute_time, data.affected_rows)
            } else if (action == 'rollback') {
              this.alertSuccess('回滚成功', id, '', data.affected_rows)
            } else if (action == 'approve') {
              this.alertSuccess('审批通过', id, '')
            } else if (action == 'disapprove') {
              this.alertSuccess('审批驳回', id, '')
            }
          } else {
            this.alertWarning('任务失败', id)
          } 
          this.handleGetSqlList()
        })
      },

      cancel () {
        Message.info('Clicked cancel');
      },

      pageChange (page) {
        this.getParams.page = page
        this.handleGetSqlList()
      },

      sizeChange(size){
        this.getParams.pagesize = size
        this.handleGetSqlList()
      },

      dateChange (data) {
        if (data[0]) {
          this.getParams.daterange = data[0] + ',' + addDate(data[1], 1)
          this.handleGetSqlList()
        }
      },

      dateClear (data) {
        console.log(data)
      }

    },
  }
</script>
