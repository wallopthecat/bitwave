<template>
  <div>
    <!-- Stream Title, Status -->
    <v-toolbar
      class="elevation-2"
      color="accentwave"
      dense
    >
      <!-- Live / Replay / Offline Indicator -->
      <v-chip
        class="flex-shrink-0"
        :class="{ blink: live }"
        :color="live ? 'red' : replay ? 'blue' : 'grey'"
        label
        outlined
        small
      >
        <v-icon
          v-show="live"
          left
          size="10"
          class="mr-2"
        >lens</v-icon>
        {{ live ? 'LIVE' : replay ? 'REPLAY' : 'offline' }}
      </v-chip>

      <!-- Stream Title -->
      <h3 class="mx-2 flex-grow-1 subtitle-1 font-weight-medium text-truncate">
        {{ title }}
      </h3>

      <!-- Bottom Tabs -->
      <template #extension>
        <v-tabs
          v-model="tabData"
          background-color="transparent"
          class="no-focus"
          show-arrows
        >
          <v-tab>{{ replay ? 'Summary' : 'Description' }}</v-tab>
          <v-tab>Replays</v-tab>
          <v-tab v-if="!replay">Stream Stats</v-tab>
        </v-tabs>
      </template>
    </v-toolbar>

    <!-- Stream Actions -->
    <div class="d-flex flex-shrink-0 align-center flex-wrap px-3 py-2">
      <div
        class="caption grey--text my-2"
        :title="timestamp"
      >
        <v-icon
          v-show="replay"
          size="16"
          color="grey"
        >restore</v-icon>
        <div class="d-inline-block">
          {{ live ? 'Started Streaming: ' : replay ? 'Streamed: ' : 'Last Streamed: ' }}
        </div>
        <v-fade-transition mode="out-in">
          <div
            class="d-inline-block"
            :key="lastStreamed"
          >
            {{ lastStreamed }}
          </div>
        </v-fade-transition>
      </div>

      <v-spacer />

      <div class="d-flex">
        <!-- Edit Stream Info Dialog -->
        <edit-stream-data
          v-if="channelOwner"
          :username="username"
          :title="title"
          :description="description"
          :nsfw="nsfw"
        />

        <!-- Restream Dialog -->
        <restream-dialog
          v-if="channelOwner"
          :username="name"
          :owner="uid"
          :live="live"
        />
      </div>

      <!-- Share Stream Dialog -->
      <ShareStream :user="name" />
    </div>

    <v-divider />

    <!-- Stream Data -->
    <v-tabs-items
      v-model="tabData"
      style="background: transparent"
      touchless
    >
      <!-- Description -->
      <v-tab-item>
        <div
          id="description"
          ref="description"
          class="pa-3"
          style="min-height: 750px"
        >
          <!-- Stream Description -->
          <vue-markdown
            v-if="description && !replay"
            :source="description"
          />
        </div>
      </v-tab-item>

      <!-- Archives -->
      <v-tab-item>
        <div
          style="min-height: 500px"
        >
          <stream-archives
            :streamer="name"
          />
        </div>
      </v-tab-item>

      <!-- Debug Stream -->
      <v-tab-item
        v-if="!replay"
      >
        <div
          style="min-height: 500px"
        >
          <debug-stream
            :streamer="name"
          />
        </div>
      </v-tab-item>

    </v-tabs-items>


    <!-- Footer -->
    <simple-footer :version="version" />
  </div>
</template>

<script>
  import { mapGetters } from 'vuex';
  import { VStore } from '@/store';

  import VueMarkdown from '@/components/VueMarkdown';
  import { timeAgo } from '@/assets/js/time-ago';
  import SimpleFooter from '@/components/SubLayout/SimpleFooter';

  // Async Components - We don't expect these components to be required frequently
  const ShareStream    = async () => await import ( '@/components/Channel/ShareStream' );
  const EditStreamData = async () => await import ( '@/components/Channel/EditStreamData' );
  const StreamArchives = async () => await import ( '@/components/Channel/StreamArchives' );
  const RestreamDialog = async () => await import ( '@/components/Restream/RestreamDialog' );

  const DebugStream = async () => await import ( '@/components/Analytics/DebugStream' );

  export default {
    name: 'LiveStreamInfo',

    components: {
      SimpleFooter,
      StreamArchives,
      VueMarkdown,
      ShareStream,
      EditStreamData,
      RestreamDialog,
      DebugStream,
    },

    props: {
      name:  { type: String },
      live:  { type: Boolean },
      title: { type: String },
      nsfw:  { type: Boolean },
      description: { type: String },
      timestamp: { type: Date },
      replay: { type: Boolean },
    },

    data () {
      return {
        tabData: null,
        lastStreamed: '• • •',
        updateInterval: null,
      };
    },

    methods: {
      updateLastStreamedAt () {
        if ( this.timestamp ) {
          try {
            this.lastStreamed = timeAgo( this.timestamp );
          } catch {
            this.lastStreamed = 'now';
          }
        } else {
          this.lastStreamed = '• • •';
        }
      },
    },

    computed: {
      ...mapGetters({
        username: VStore.$getters.getUsername,
        uid: VStore.$getters.getUID,
        isStreamer : VStore.$getters.isStreamer,
        isAdmin  : VStore.$getters.isAdmin,
        getFeatureFlag : VStore.$getters.getFeatureFlag,
      }),

      featureFlag () {
        const flags = this.getFeatureFlag( 'streamStats' );
        if ( flags._disabled ) return false;
        if ( flags._enabled ) return true;
        if ( flags.users && this.uid ) return true;
        if ( flags.streamers && this.isStreamer ) return true;
        if ( flags.owner && this.channelOwner ) return true;
        if ( flags.admin && this.isAdmin ) return true;
        return false;
      },

      channelOwner () {
        if ( !this.username || !this.name ) return false;
        return this.name.toLowerCase() === this.username.toLowerCase();
      },

      version () {
        return `v${process.env.version}`;
      },
    },

    watch: {
      timestamp: function ( val, oldVal ) {
        this.updateLastStreamedAt();
      }
    },

    mounted () {
      this.updateLastStreamedAt();
      this.updateInterval = setInterval( this.updateLastStreamedAt, 60 * 1000 );
    },

    beforeDestroy() {
      if ( this.updateInterval ) clearInterval( this.updateInterval );
    }
  };
</script>

<style lang="scss">
  .v-tabs.no-focus {
    .v-tab--active:hover::before,
    .v-tab--active::before {
      opacity: 0;
    }
  }
</style>
