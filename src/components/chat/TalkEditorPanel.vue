<template>
  <div>
    <el-container class="panel-container">
      <!-- 头部信息 -->
      <el-header class="panel-header">
        <div class="title">
          <span class="badge friend" v-if="params.source == 1">友</span>
          <span class="badge" v-else>群</span>
          <span class="name">
            <span
              v-if="params.source == 1"
              @click="catFriendDetail(params.receiveId)"
              >{{ params.nickname }}</span
            >
            <span v-else @click="groupBoxShow = !groupBoxShow">{{
              params.nickname
            }}</span>
            <span class="num" v-show="groupNum && params.source == 2"
              >({{ groupNum }}人)</span
            >
          </span>
        </div>
        <div class="online" v-show="params.source == 1">
          <p class="badge" :class="{ 'yes-badge': isOnline == true }">
            <span
              class="badge-status-dot badge-status-processing"
              v-show="isOnline == true"
            ></span>
            <span>{{ isOnline ? "在线" : "离线" }}</span>
          </p>
          <p class="input" v-show="keyEvent.isShow">对方正在输入 ...</p>
        </div>
        <div class="means">
          <el-tooltip content="历史消息" placement="top">
            <p>
              <i class="el-icon-time" @click="findChatRecord = true"></i>
            </p>
          </el-tooltip>
          <el-tooltip content="群公告" placement="top">
            <p v-show="params.source == 2">
              <i
                class="iconfont icon-gonggao2"
                @click="isShowGroupNotice = true"
              ></i>
            </p>
          </el-tooltip>
          <el-tooltip content="群设置" placement="top">
            <p v-show="params.source == 2">
              <i
                class="el-icon-setting"
                @click="groupBoxShow = !groupBoxShow"
              ></i>
            </p>
          </el-tooltip>
        </div>
      </el-header>

      <!-- 主体信息 -->
      <el-main
        class="no-padding panel-main lum-scrollbar"
        id="lumenChatPanel"
        @scroll.native="talkPanelScroll($event)"
      >
        <div class="talk-container">
          <div class="toolbars no-select">
            <span v-show="loadRecord.status == 0">
              <i class="el-icon-loading"></i> 正在加载数据中...
            </span>
            <span v-show="loadRecord.status == 1" @click="loadChatRecords"
              ><i class="el-icon-bottom"></i> 查看更多消息...</span
            >
            <span v-show="loadRecord.status == 2" class="not-have-msg"
              >没有更多消息了...</span
            >
          </div>
        </div>

        <!-- 遍历对话记录 -->
        <div v-for="(item, idx) in records" :key="item.id">
          <!-- 群消息 -->
          <div v-if="item.msg_type == 3" class="talk-container pd-t5">
            <div
              v-if="item.invite.type == 1 || item.invite.type == 3"
              class="system-msg invite-tip"
            >
              <a @click="catFriendDetail(item.invite.operate_user.id)">{{
                item.invite.operate_user.nickname
              }}</a>
              <span>{{ item.invite.type == 1 ? "邀请了" : "将" }}</span>
              <template v-for="(user, uidx) in item.invite.users">
                <a @click="catFriendDetail(user.id)">{{ user.nickname }}</a>
                <em v-show="uidx < item.invite.users.length - 1">、</em>
              </template>
              <span>{{
                item.invite.type == 1 ? "加入了群聊" : "踢出了群聊"
              }}</span>
            </div>
            <div v-else-if="item.invite.type == 2" class="system-msg">
              <a @click="catFriendDetail(item.invite.operate_user.id)">{{
                item.invite.operate_user.nickname
              }}</a>
              <span>退出了群聊</span>
            </div>
          </div>

          <!-- 撤回消息 -->
          <div v-else-if="item.is_revoke == 1" class="talk-container pd-t5">
            <div class="system-msg">
              <span v-if="$store.state.user.uid == item.user_id"
                >你撤回了一条消息 | {{ sendTime(item.created_at) }}</span
              >
              <span v-else-if="params.source == 1"
                >对方撤回了一条消息 | {{ sendTime(item.created_at) }}</span
              >
              <span v-else
                >"{{ item.nickname }}" 撤回了一条消息 |
                {{ sendTime(item.created_at) }}</span
              >
            </div>
          </div>

          <div
            v-else
            class="talk-container"
            :class="{
              'direction-rt': item.float == 'right',
              'open-checkbox': multiSelect.isOpen === true,
            }"
          >
            <aside class="checkbox-column">
              <i
                class="el-icon-success"
                :class="{ selected: verifyMultiSelect(item.id) }"
                @click="triggerMultiSelect(item.id)"
              ></i>
            </aside>
            <aside class="avatar-column">
              <div class="avatar">
                <img
                  :src="item.avatar"
                  @click="catFriendDetail(item.user_id)"
                  :onerror="$store.state.detaultAvatar"
                  width="30"
                  height="30"
                />
              </div>
            </aside>
            <div class="content-column">
              <div
                class="talktime"
                v-show="!(item.source == 2 && item.float == 'left')"
                v-text="parseTime(item.created_at, '{m}月{d}日 {h}:{i}')"
              />
              <div
                class="subtitle"
                v-if="item.source == 2 && item.float == 'left'"
              >
                <span class="nickname" v-text="item.nickname"></span>
                <span v-show="item.friend_remarks"
                  >({{ item.friend_remarks }})</span
                >
                <span class="talktime2">{{
                  parseTime(item.created_at, "{m}月{d}日 {h}:{i}")
                }}</span>
              </div>
              <div class="talkbox">
                <!-- 文本消息角标 -->
                <div class="arrow" v-show="item.msg_type == 1"></div>

                <!-- 文本消息 -->
                <text-message
                  v-if="item.msg_type == 1"
                  :content="item.content"
                  :float="item.float"
                  :full-width="false"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 图片消息 -->
                <image-message
                  v-else-if="item.msg_type == 2 && item.file.file_type == 1"
                  :src="item.file.file_url"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 音频文件预留 -->
                <audio-message
                  v-else-if="item.msg_type == 2 && item.file.file_type == 2"
                  :src="item.file.file_url"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 视频文件预留 -->
                <video-message
                  v-else-if="item.msg_type == 2 && item.file.file_type == 3"
                  :src="item.file.file_url"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 文件消息 -->
                <file-message
                  v-else-if="item.msg_type == 2 && item.file.file_type == 4"
                  :file="item.file"
                  :record_id="item.id"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 会话记录消息 -->
                <forward-message
                  v-else-if="item.msg_type == 4"
                  :forward="item.forward"
                  :record_id="item.id"
                  @contextmenu.native="onCopy(idx, item, $event)"
                />

                <!-- 代码块消息 -->
                <div v-else-if="item.msg_type == 5">
                  <code-message
                    :code="item.code_block.code"
                    :lang="item.code_block.code_lang"
                    @contextmenu.native="onCopy(idx, item, $event)"
                  />
                </div>

                <!-- 未知消息 -->
                <div class="unknown-msg" v-else>
                  未知消息类型[{{ item.msg_type }}]
                </div>
              </div>
            </div>
          </div>

          <div
            class="talk-container pd-t5"
            v-show="compareTime(idx, item.created_at)"
          >
            <div class="talk-time" v-text="sendTime(item.created_at)"></div>
          </div>
        </div>
      </el-main>

      <!-- 页脚信息 -->
      <el-footer class="panel-footer" height="160">
        <template v-if="!multiSelect.isOpen">
          <me-editor
            ref="talkEditor"
            @send="submitSendMesage"
            @sendCodeBlock="sendCodeBlock"
            @keyboard-event="keyboardEvent"
          />
        </template>
        <template v-else>
          <div class="multi-select">
            <div class="multi-title" v-show="multiSelect.items.length > 0">
              <span>已选中：{{ multiSelect.items.length }} 条消息</span>
            </div>
            <div class="multi-btn-group">
              <div class="multi-icon" @click="handleMultiMode(2)">
                <i class="el-icon-position"></i>
              </div>
              <p>合并转发</p>
            </div>
            <div class="multi-btn-group">
              <div class="multi-icon" @click="handleMultiMode(3)">
                <i class="el-icon-delete"></i>
              </div>
              <p>批量删除</p>
            </div>
            <div class="multi-btn-group">
              <div class="multi-icon" @click="closeMultiSelect">
                <i class="el-icon-close"></i>
              </div>
              <p>关闭</p>
            </div>
          </div>
        </template>
      </el-footer>

      <!-- 群设置侧边栏 -->
      <div
        class="sidebar-box"
        :class="{ show: groupBoxShow }"
        v-outside="hideChatGroup"
      >
        <group-panel
          v-if="params.source == 2"
          :group-id="params.receiveId"
          @close="hideChatGroup"
          @send-group="hideChatGroup"
          @quit-group="quitGroupSuccess"
          @disturb-change="disturbChange"
          @group-info="syncGroupInfo"
        />
      </div>

      <!-- 置底按钮 -->
      <transition name="el-fade-in-linear">
        <div
          class="tips-board"
          v-show="tipsBoard"
          @click="talkPanelScrollBottom"
        >
          <svg-mention-down style="width: 10px; height: 10px; color: black" />
          <span>回到底部</span>
        </div>
      </transition>
    </el-container>

    <!-- 消息管理器 -->
    <transition name="el-fade-in-linear">
      <talk-search-record
        v-if="findChatRecord"
        @close="findChatRecord = false"
        :source="params.source"
        :receive-id="params.receiveId"
        :titleName="params.nickname"
      />
    </transition>

    <!-- 选择联系人窗口 -->
    <transition name="el-fade-in-linear">
      <user-contacts
        v-if="selectContacts.isShow"
        @close="selectContacts.isShow = false"
        @confirm="confirmSelectContacts"
      />
    </transition>

    <!-- 查看好友用户信息 -->
    <user-business-card ref="userBusinessCard" />

    <!-- 群公告组件 -->
    <transition name="el-fade-in-linear">
      <group-notice
        v-if="isShowGroupNotice"
        :group-id="params.receiveId"
        @close="isShowGroupNotice = false"
      />
    </transition>
  </div>
</template>
<script>
import Vue from "vue";
import UserBusinessCard from "@/components/user/UserBusinessCard";
import TalkSearchRecord from "@/components/chat/TalkSearchRecord";
import UserContacts from "@/components/chat/UserContacts";
import GroupPanel from "@/components/group/GroupPanel";
import GroupNotice from "@/components/group/GroupNotice";
import MeEditor from "@/components/editor/MeEditor";
import { SvgMentionDown } from "@/core/icons";
import {
  ServeTalkRecords,
  ServeForwardRecords,
  ServeRemoveRecords,
  ServeRevokeRecords,
} from "@/api/chat";
import { ServeCollectEmoticon } from "@/api/emoticon";
import {
  formateTime,
  parseTime,
  copyTextToClipboard,
  replaceEmoji,
} from "@/utils/functions";

export default {
  name: "TalkEditorPanel",
  components: {
    MeEditor,
    UserContacts,
    GroupPanel,
    TalkSearchRecord,
    UserBusinessCard,
    GroupNotice,
    SvgMentionDown,
  },
  props: {
    //对话相关参数
    params: {
      type: Object,
      default: {
        //消息来源（1：好友私信 2:群聊）
        source: 0,
        //消息接收者ID（好友ID或者群聊ID）
        receiveId: 0,
        nickname: "",
      },
    },

    //用户是否在线
    isOnline: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      groupNum: 0,

      //查看聊天记录信息
      forwardRecordBox: {
        isShow: false,
        records_id: 0,
      },

      //记录加载相关参数
      loadRecord: {
        status: 0,
        minRecord: 0,
        scrollHeight: 0,
      },

      //多选相关操作
      multiSelect: {
        isOpen: false,
        items: [],
        mode: 0,
      },

      //选择联系人窗口
      selectContacts: {
        isShow: false,
      },

      //图片查看器列表
      images: [],
      groupBoxShow: false,

      //键盘输入事件
      keyEvent: {
        isShow: false,
        time: 0,
      },

      //聊天记录管理器数据
      findChatRecord: false,

      // 置底按钮是否显示
      tipsBoard: false,

      isShowGroupNotice: false,
    };
  },
  computed: {
    records() {
      return this.$root.message.records;
    },
    inputEvent() {
      return this.$store.state.notify.inputEvent;
    },
    scroll() {
      return this.$store.state.notify.scroll;
    },
  },
  watch: {
    // 监听面板传递参数
    params(val) {
      this.loadRecord.minRecord = 0;
      this.$root.message.records = [];
      this.multiSelect = {
        isOpen: false,
        items: [],
        mode: 0,
      };

      this.loadChatRecords();
      this.tipsBoard = false;
    },
    scroll(n, o) {
      this.$nextTick(() => {
        this.talkPanelScrollBottom();
      });
    },
    //监听好友键盘事件
    inputEvent(n, o) {
      this.keyEvent.isShow = true;
      setTimeout(() => {
        this.keyEvent.isShow = false;
      }, 2000);
    },
  },
  mounted() {
    this.loadChatRecords();
  },
  methods: {
    //聊天时间人性化处理
    sendTime: formateTime,
    parseTime: parseTime,

    //发送消息方法
    sendSocket(message) {
      this.$root.socket.send(message);
    },

    //回车键发送消息回调事件
    submitSendMesage(content) {
      //调用父类Websocket组件发送消息
      this.sendSocket({
        event: "event_talk",
        data: {
          // 发送消息的用户ID
          send_user: this.$store.state.user.uid,

          // 接受者消息ID(用户ID或群ID)
          receive_user: this.params.receiveId,

          // 聊天类型  1:私聊 2:群聊信息显示用户昵称
          source_type: this.params.source,

          // 消息文本
          text_message: content,
        },
      });
    },
    // 发送代码消息
    sendCodeBlock() {},

    //推送编辑事件消息
    keyboardEvent() {
      // 判断是否推送键盘输入事件消息
      if (!this.$store.state.settings.keyboardEventNotify) {
        return false;
      }

      let time = new Date().getTime();

      //判断当前对话是否属于私聊信息
      if (this.params.source == 2 || !this.isOnline) return;

      //判断在两秒内是否已推送事件
      if (this.keyEvent.time != 0 && time - this.keyEvent.time < 2000) {
        return;
      }

      this.keyEvent.time = new Date().getTime();

      //调用父类Websocket组件发送消息
      this.sendSocket({
        event: "event_keyboard",
        data: {
          send_user: this.$store.state.user.uid,
          receive_user: this.params.receiveId,
        },
      });
    },

    //加载用户聊天详情信息
    loadChatRecords() {
      let data = {
        record_id: this.loadRecord.minRecord,
        receive_id: this.params.receiveId,
        source: this.params.source,
      };

      this.loadRecord.status = 0;
      this.loadRecord.scrollHeight = document.getElementById(
        "lumenChatPanel"
      ).scrollHeight;
      ServeTalkRecords(data)
        .then((res) => {
          //防止点击切换过快消息返回延迟，导致信息错误
          if (
            res.code != 200 ||
            (data.receive_id != this.params.receiveId &&
              data.source != this.params.source)
          ) {
            return;
          }

          let records = data.record_id == 0 ? [] : this.$root.message.records;

          records.unshift(...res.data.rows.reverse());

          this.loadRecord.minRecord =
            res.data.rows.length == res.data.limit ? res.data.record_id : 0;

          let user_id = this.$store.state.user.uid;

          this.$root.message.records = records.map((item) => {
            item.float =
              item.user_id == 0
                ? "center"
                : item.user_id == user_id
                ? "right"
                : "left";

            if (item.msg_type == 1) {
              item.content = replaceEmoji(item.content);
            }

            return item;
          });

          this.loadRecord.status =
            res.data.rows.length >= res.data.limit ? 1 : 2;

          //滚动条处理
          let el = document.getElementById("lumenChatPanel");
          this.$nextTick(function () {
            if (data.record_id == 0) {
              //首页加载数据滚动条置底
              el.scrollTop = el.scrollHeight;
            } else {
              //加载数据完成之后将滚动条重置到加载之前的位置
              el.scrollTop = el.scrollHeight - this.loadRecord.scrollHeight;
            }
          });
        })
        .catch((e) => {
          this.loadRecord.status = 1;
        });
    },

    //多选处理方式
    handleMultiMode(type) {
      if (this.multiSelect.items.length <= 1) {
        return false;
      }

      this.multiSelect.mode = type;
      if (type == 1) {
        //逐条转发
        if (this.verifyMultiSelectType(4)) {
          this.$notify({
            title: "消息转发",
            message: "会话记录不支持合并转发",
          });
          return false;
        }

        this.selectContacts.isShow = true;
      } else if (type == 2) {
        //合并转发
        if (this.verifyMultiSelectType(4)) {
          this.$notify({
            title: "消息转发",
            message: "会话记录不支持合并转发",
          });
          return false;
        }

        this.selectContacts.isShow = true;
      } else {
        //批量删除
        let ids = this.multiSelect.items;
        ServeRemoveRecords({
          source: this.params.source,
          receive_id: this.params.receiveId,
          record_id: ids.join(","),
        }).then((res) => {
          if (res.code == 200) {
            this.delRecords(ids).closeMultiSelect();
          }
        });
      }
    },

    //确认消息转发联系人事件
    confirmSelectContacts(arr) {
      let user_ids = [],
        group_ids = [];
      arr.forEach((item) => {
        if (item.type == 1) {
          user_ids.push(item.id);
        } else {
          group_ids.push(item.id);
        }
      });

      this.selectContacts.isShow = false;
      ServeForwardRecords({
        forward_mode: this.multiSelect.mode,
        source: this.params.source,
        receive_id: this.params.receiveId,
        records_ids: this.multiSelect.items,
        receive_user_ids: user_ids,
        receive_group_ids: group_ids,
      }).then((res) => {
        if (res.code == 200) {
          this.closeMultiSelect();
          this.$notify({
            title: "消息转发",
            message: "消息转发成功...",
            type: "success",
          });
        }
      });
    },

    //处理消息时间是否显示
    compareTime(index, datetime) {
      if (datetime == undefined) {
        return false;
      }

      if (this.$root.message.records[index].is_revoke == 1) {
        return false;
      }

      datetime = datetime.replace(/-/g, "/");
      let time = Math.floor(Date.parse(datetime) / 1000);
      let currTime = Math.floor(new Date().getTime() / 1000);

      //当前时间5分钟内时间不显示
      if (currTime - time < 300) return false;

      //判断是否是最后一条消息,最后一条消息默认显示时间
      if (index == this.$root.message.records.length - 1) {
        return true;
      }

      let nextDate = this.$root.message.records[index + 1].created_at.replace(
        /-/g,
        "/"
      );

      return !(
        parseTime(new Date(datetime), "{y}-{m}-{d} {h}:{i}") ==
        parseTime(new Date(nextDate), "{y}-{m}-{d} {h}:{i}")
      );
    },

    //查看好友用户信息
    catFriendDetail(value) {
      this.$refs.userBusinessCard.open(value);
    },

    //撤回消息
    revokeRecords(idx, item) {
      ServeRevokeRecords({
        record_id: item.id,
      }).then((res) => {
        if (res.code == 200) {
          this.$root.message.records[idx].is_revoke = 1;
        }
      });
    },

    //删除消息
    removeRecords(idx, item) {
      let user_id = this.$store.state.user.uid;
      let receive_id = item.receive_id;
      if (item.source == 1 && item.user_id != user_id) {
        receive_id = item.user_id;
      }

      ServeRemoveRecords({
        source: item.source,
        receive_id: receive_id,
        record_id: item.id,
      }).then((res) => {
        if (res.code == 200) {
          this.$delete(this.$root.message.records, idx);
        }
      });
    },

    //转发消息
    forwardRecords(idx, item) {
      alert("单条记录转发开发中...");
    },

    //从列表中删除记录
    delRecords(arr) {
      arr.forEach((record_id) => {
        let i = this.$root.message.records.findIndex(
          (item) => item.id == record_id
        );
        if (i >= 0) {
          this.$delete(this.$root.message.records, i);
        }
      });

      return this;
    },

    //开启多选模式
    openMultiSelect() {
      this.multiSelect.isOpen = true;
    },

    //关闭多选模式
    closeMultiSelect() {
      this.multiSelect.isOpen = false;
      this.multiSelect.items = [];
    },

    //判断记录是否选中
    verifyMultiSelect(records_id) {
      return this.multiSelect.items.indexOf(records_id) >= 0;
    },

    //触发多选事件
    triggerMultiSelect(records_id) {
      let i = this.multiSelect.items.indexOf(records_id);
      if (i >= 0) {
        this.multiSelect.items.splice(i, 1);
      } else {
        if (this.multiSelect.items.length >= 30) {
          this.$notify({
            title: "温馨提示",
            message: "批量操作最大支持30条数据...",
          });
          return false;
        }
        this.multiSelect.items.push(records_id);
      }
    },

    //验证是否存在选择的指定类型的消息
    verifyMultiSelectType(type) {
      for (let o of this.$root.message.records) {
        if (this.verifyMultiSelect(o.id) && o.msg_type == type) {
          return true;
        }
      }
      return false;
    },

    //消息点击右键触发自定义菜单
    onCopy(idx, item, event) {
      let menus = [];
      let content = "";
      if (document.getElementById("copy_class_" + item.id)) {
        content = document.getElementById("copy_class_" + item.id).innerText;
      }

      if (content) {
        menus.push({
          label: "复制",
          icon: "el-icon-document-copy",
          customClass: "cus-contextmenu-item",
          onClick: () => {
            copyTextToClipboard(content);
          },
        });
      }

      if (item.user_id == this.$store.state.user.uid) {
        let time =
          new Date().getTime() - Date.parse(item.created_at.replace(/-/g, "/"));
        if (Math.floor(time / 1000 / 60) < 2) {
          menus.push({
            label: "撤回",
            icon: "el-icon-s-flag",
            customClass: "cus-contextmenu-item",
            onClick: () => {
              this.revokeRecords(idx, item);
            },
          });
        }
      }

      menus.push({
        label: "删除",
        icon: "el-icon-delete",
        customClass: "cus-contextmenu-item",
        onClick: () => {
          this.removeRecords(idx, item);
        },
      });

      menus.push({
        label: "转发",
        icon: "el-icon-s-promotion",
        customClass: "cus-contextmenu-item",
        onClick: () => {
          this.forwardRecords(idx, item);
        },
      });

      menus.push({
        label: "引用",
        icon: "el-icon-connection",
        customClass: "cus-contextmenu-item",
        onClick: () => {},
      });

      menus.push({
        label: "多选",
        icon: "el-icon-finished",
        customClass: "cus-contextmenu-item",
        onClick: () => {
          this.openMultiSelect();
        },
      });

      // 判断是否是图片消息
      if (item.msg_type == 2 && item.file.file_type == 1) {
        menus.push({
          label: "收藏",
          icon: "el-icon-picture",
          customClass: "cus-contextmenu-item",
          onClick: () => {
            ServeCollectEmoticon({
              record_id: item.id,
            })
              .then((res) => {
                if (res.code == 200) {
                  this.$notify({
                    title: "收藏提示",
                    message: "表情包收藏成功...",
                    type: "success",
                  });

                  this.$refs.talkEditor.reloadEmoticon();
                }
              })
              .catch((err) => {
                this.$notify({
                  title: "收藏提示",
                  message: "表情包收藏失败...",
                  type: "warning",
                });
              });
          },
        });
      }

      this.$contextmenu({
        items: menus,
        event,
        customClass: "cus-contextmenu",
        zIndex: 3,
        minWidth: 120,
      });

      this.closeMultiSelect();
      event.preventDefault();
    },
    hideChatGroup() {
      this.groupBoxShow = false;
    },

    //修改群聊免打扰状态
    disturbChange(detail) {
      let key = this.$store.state.talks.items.findIndex(
        (item) => item.index_name == `2_${this.params.receiveId}`
      );
      if (key == -1) return false;
      this.$store.commit({
        type: "UPDATE_TALK_ITEM",
        key: key,
        item: {
          not_disturb: parseInt(detail.status),
        },
      });
    },

    //退出群聊回调事件
    quitGroupSuccess() {
      this.$emit("close-talk");
    },

    //同步群信息
    syncGroupInfo(groupInfo) {
      this.groupNum = groupInfo.members_num;
    },

    // 对话面板滚动事件
    talkPanelScroll(e) {
      this.tipsBoard = !(
        Math.ceil(e.target.scrollTop) + e.target.clientHeight >=
        e.target.scrollHeight
      );
    },

    // 聊天版本滚动到底部
    talkPanelScrollBottom() {
      let el = document.getElementById("lumenChatPanel");
      el.scrollTop = el.scrollHeight;
    },
  },
};
</script>
<style lang="less" scoped>
@import "~@/assets/css/talk/style.less";
@import "~@/assets/css/talk/panel-record.less";
</style>