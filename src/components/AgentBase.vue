<template>
  <el-container>
    <el-menu
      ref="menu"
      style="-webkit-user-select:none;"
      mode="vertical"
      id="side-menu"
      background-color="#545c64"
      text-color="#fff"
      active-text-color="#ffd04b"
      router>
      <el-menu-item-group :index="group" v-for="(group_modules, group) in matching_modules_grouped" v-bind:key="group">
        <span class="menu-title" slot="title">{{group}}</span>
        <el-menu-item :index="m.path" v-for="m in group_modules" v-bind:key="m.path" :route="{name: m.path}">
          <i v-bind:class="m.icon"></i>
          <span>{{m.label}}</span>
        </el-menu-item>
      </el-menu-item-group>
    </el-menu>
    <nav id="top-bar" class="navbar navbar-expand-lg navbar-light bg-light">
      <a class="navbar-brand" href="#">{{connection.label}}</a>
      <el-form
        :disabled="dids.length === 0">
        <el-select
          v-model="active_did"
          filterable placeholder="Activate DID">
          <el-option
            v-for="did in dids"
            :key="did.did"
            :label="did.did"
            :value="did">
          </el-option>
        </el-select>
        <!--
          <el-select
          v-if="$refs.ledgerTab"
          v-model="$refs.ledgerTab.active_ledger_selector.ledger"
          filterable placeholder="activate ledger" >
          <el-option
          v-for="ledger in $refs.ledgerTab.ledgers"
          :key="ledger.name"
          :label="ledger.name"
          :value="ledger.name">
          </el-option>
          </el-select>
        -->
      </el-form>
    </nav>
    <taa></taa>

    <el-main id="main-display">
      <router-view></router-view>
    </el-main>
  </el-container>
</template>

<script>
const bs58 = require('bs58');

import Vue from 'vue';
import { mapState, mapActions } from "vuex";
import { from_store } from '../connection_detail.ts';
import message_bus from '@/message_bus.ts';
import share, {share_source} from '@/share.ts';
import components, {shared} from './components.ts';
import Taa from './TAA.vue';

// The (. && .. && ...) || 'default' syntax provides defaults for modules that lack any level of the metadata
//  definition. It would be useful if javascript had an Elvis Operator, but it does not.
// The icons from https://element.eleme.io/#/en-US/component/icon
let module_list = Object.entries(components).map(([modulename, module]) => ({
  path: module.default.name,
  label: (module.metadata && module.metadata.menu && module.metadata.menu.label) || module.default.name,
  icon: (module.metadata && module.metadata.menu && module.metadata.menu.icon) || 'el-icon-document',
  group: (module.metadata && module.metadata.menu && module.metadata.menu.group) || 'Other',
  priority: (module.metadata && module.metadata.menu && module.metadata.menu.priority) || 100,
  dev_only: (module.metadata && module.metadata.menu && module.metadata.menu.dev_only) || false,
  required_protocols: (module.metadata && module.metadata.menu && module.metadata.menu.required_protocols) || [],
}));

//sort modules by priority, then label
module_list.sort((a,b)=> a.priority - b.priority ||
                  a.label.localeCompare(b.label));

export default {
  name: 'agent-base',
  mixins: [
    message_bus({ events: {
      'redirect': (vm, route) => {
        vm.redirect(route);
      },
      'send-message': (v, msg, return_route) => {
        v.send_connection_message(msg, return_route);
      },
      'did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/notification/1.0/problem-report':
      (vm, msg) => {
        vm.$notify.error({
          title: 'Small problem...',
          message: (text => {
            if (text.length > 30) {
              return text.slice(0, 30).trim() + '...';
            }
            return text;
          })(msg['explain-ltxt']),
          onClick: () => vm.redirect('message-history'),
          duration: 4000,
          customClass: 'problem-report-notification'
        })
      }
    }}),
    share_source(shared),
    share({
      use: ['dids', 'public_did', 'protocols', 'is_dev_interface'],
      actions: [
        'fetch_dids',
        'fetch_active_did',
        'activate_did',
        'fetch_protocols',
        'fetch_connections'
      ]
    })
  ],
  props: ['agentid'],
  components: {Taa},
  data: function() {
    return {
      'connection': {'label':'loading...'},
      'modules': module_list,
      'return_route_poll_timer': '',
    }
  },
  computed: {
    active_did: {
      get () {
        return this.public_did || "";
      },
      set (did) {
        return this.activate_did(did)
      }
    },
    matching_modules_grouped: function(){
      // This computed attribute updates when the supported protocol list is updated.
      let pid_list = this.protocols.map(p => p.pid);

      let view_filtered_list = module_list
      if(!this.is_dev_interface) {
        view_filtered_list = module_list.filter(m => !m.dev_only)
      }
      let filtered_list = view_filtered_list.filter(function(m){
        return m.required_protocols.every(req_protocol => pid_list.includes(req_protocol));
      });
      //group modules
      let module_groups = filtered_list.reduce(function (r, m) {
          r[m.group] = r[m.group] || [];
          r[m.group].push(m);
          return r;
      }, Object.create(null));
      return module_groups;
    }

  },
  methods: {
    ...mapActions("Agents", ["get_agent"]),
    async send_connection_message(msg){
      await this.connection_loaded;
      this.connection.send_message(msg);
    },
    async processInbound(msg){
      // RFC 0348 Step 1: modify prefix (if present) to old standard
      let OLD = "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/";
      let NEW = "https://didcomm.org/";
      if(msg['@type'].startsWith(NEW)){
        msg['@type'] = msg['@type'].replace(NEW, OLD);
      }
      // END RFC 0348 Step 1
      this.$message_bus.$emit('message-received', msg);
      this.$message_bus.$emit(msg['@type'], msg);
    },
    return_route_poll(){
      // This brute forces picking up return route messages
      let msg = {
        "@type": "did:sov:BzCbsNYhMrjHiqZDTUASHg;spec/trust_ping/1.0/ping",
        "response_requested": true,
        "~transport": {
          "return_route": "all"
        }
      };
      this.send_message(msg);
    },
    redirect: function(route) {
      this.$router.push({name: route});
      this.$refs.menu.updateActiveIndex(route);
    }
  },
  created: async function() {
    this.connection_loaded = (async () => {
      this.connection = from_store(await this.get_agent(this.agentid), this.processInbound);
    })();
    await this.connection_loaded;
    this.fetch_protocols();
    this.fetch_dids();
    this.fetch_active_did();
    this.fetch_connections();
    this.$message_bus.$emit('agent-created');
    //start poll timer
    if(this.connection.needs_return_route_poll()){
      this.return_route_poll_timer = setInterval(this.return_route_poll, 10000);
    }
  },
  beforeDestroy: function() {
    if(this.connection.needs_return_route_poll()) {
      clearInterval(this.return_route_poll_timer)
    }
  }
}
</script>

<style>
  #top-bar {
    position:absolute;
    right: 0px;
    left:200px;
    height: 60px;
  }

  #main-display {
    position: absolute;
    right:0px;
    bottom: 0px;
    top: 60px;
    left: 200px;
  }

  #side-menu {
    height: 100%;
    width: 200px;
    position: absolute;

  }
  #side-menu .menu-title {
    color: #fff
  }
  #side-menu i {
    color: #409EFF;
  }

  .el-menu-item-group__title {
    font-size: 18px;
  }

  .el-menu-item {
    height: 36px;
    line-height: 36px;
  }

  .problem-report-notification {
    cursor: pointer;
  }

</style>
