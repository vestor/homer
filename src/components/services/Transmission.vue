<template>
  <Generic :item="item">
    <template #content>
      <p class="title is-4">{{ item.name }}</p>
      <p class="subtitle is-6">
        <span v-if="error" class="error">An error has occurred.</span>
        <template v-else>
          <span class="down">
            <i class="fas fa-download"></i> {{ downRate }}
          </span>
          <span class="up"> <i class="fas fa-upload"></i> {{ upRate }} </span>
        </template>
      </p>
    </template>
    <template #indicator>
      <span v-if="!error" class="count"
        >{{ count }}
        <template v-if="count === 1">torrent</template>
        <template v-else>torrents</template>
      </span>
    </template>
  </Generic>
</template>

<script>
import Generic from "./Generic.vue";

// Units to add to download and upload rates.
const units = ["B", "kiB", "MiB", "GiB"];

// Take the rate in bytes and keep dividing it by 1k until the lowest
// value for which we have a unit is determined. Return the value with
// up to two decimals as a string and unit/s appended.
const displayRate = (rate) => {
  let i = 0;

  while (rate > 1000 && i < units.length) {
    rate /= 1000;
    i++;
  }

  return (
    Intl.NumberFormat(undefined, { maximumFractionDigits: 2 }).format(
      rate || 0
    ) + ` ${units[i]}/s`
  );
};

export default {
  name: "Transmission",
  props: { item: Object },
  components: { Generic },
  // Properties for download, upload, torrent count and errors.
  data: () => ({ dl: null, ul: null, count: null, error: null }),
  // Computed properties for the rate labels.
  computed: {
    downRate: function () {
      return displayRate(this.dl);
    },
    upRate: function () {
      return displayRate(this.ul);
    },
  },
  created() {
    // Set intervals if configured so the rates and/or torrent count
    // will be updated.
    const interval = parseInt(this.item.interval, 10) || 0;

    if (interval > 0) {
      setInterval(() => this.getStats(), interval);
    }

    // Fetch the initial values.
    this.getStats();
  },
  methods: {

    // Perform a call to the JSON-RPC service and parse the response
    // as JSON and stats, which is then returned.
    getStats: async function () {
      const headers = { "Content-Type": "application/json" };

      if (this.item.username && this.item.password) {
        headers[
          "Authorization"
        ] = `${this.item.username}:${this.item.password}`;
      }

      return fetch(`${this.item.xmlrpc.replace(/\/$/, "")}/transmission/rpc`, {
        method: "POST",
        headers,
        body: `{method:"session-stats"}`,
      })
        .then((response) => {
          if (!response.ok) {
            throw Error(response.statusText);
          }
          return response.text();
        })
        .then((text) =>
          Promise.resolve(new JSON().parse(text)["arguments"])
        )
        .then((json) => {
            this.dl = json["downloadSpeed"]
            this.ul = json["uploadSpeed"]
            this.count = json["torrentCount"]
        })
    },
  },
};
</script>

<style scoped lang="scss">
.error {
  color: #e51111 !important;
}
.down {
  margin-right: 1em;
}
.count {
  color: var(--text);
  font-size: 0.8em;
}
</style>
