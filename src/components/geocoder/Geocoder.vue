<template>

  <v-toolbar-items>
    <v-combobox
      return-object
      :no-filter="noFilter"
      v-model="selected"
      :autofocus="autofocus"
      :items="resultItems"
      :label="placeHolder"
      append-icon=""
      :dark="dark"
      :color="dark ? 'white': ''"
      :persistent-hint="persistentHint"
      :hidden="hideSearch"
      :rounded="rounded"
      :search-input.sync="search"
    ></v-combobox>

    <v-btn @click='toggle()' icon :dark="dark" slot="activator">
      <v-icon medium>{{buttonIcon}}</v-icon>
    </v-btn>
  </v-toolbar-items>

</template>

<script>
  import {Mapable} from '../../mixins/Mapable';
  import {GeocoderController} from './GeocoderController';
  import {applyTransform} from 'ol/extent';
  import {getTransform, fromLonLat} from 'ol/proj';

  export default {
    name: 'wgu-geocoder-input',
    mixins: [Mapable],
    props: {
      buttonIcon: {type: String, required: false, default: 'search'},
      rounded: {type: Boolean, required: false, default: true},
      autofocus: {type: Boolean, required: false, default: true},
      dark: {type: Boolean, required: false, default: false},
      persistentHint: {type: Boolean, required: false, default: true}
    },
    data () {
      return {
        placeHolder: '',
        results: [],
        lastQueryStr: '',
        noFilter: true,
        search: null,
        selecting: false,
        selected: null,
        hideSearch: true,
        timeout: null
      }
    },
    computed: {
      resultItems () {
        let items = [];
        if (!this.results) {
          return items;
        }
        this.trace(`computed.resultItems() - cur results len=${this.results.length}`);

        // Convert results to v-combobox (text, value) Items
        this.results.forEach(result => {
          this.trace(`add to this.items: ${result.address.name}`);
          items.push({text: result.address.name, value: result});
        });

        return items;
      }
    },
    methods: {
      trace (str) {
        this.debug && console && console.info(str);
      },
      toggle () {
        // Show/hide search combobox
        this.hideSearch = !this.hideSearch
      },
      // Query by string - should return list of selection items (adresses) for ComboBox
      querySelections (queryStr) {
        this.timeout = setTimeout(() => {
          // Let Geocoder Provider do the query
          // items (item.text fields) will be shown in combobox dropdown suggestions
          this.trace(`geocoderController.query: ${queryStr}`);
          this.geocoderController.query(queryStr)
            .then(results => this.onQueryResults(results))
            .catch(err => this.onQueryError(err))
        }, this.queryDelay);
      },
      onQueryResults (results) {
        // Handle query results from GeocoderController
        this.timeout && clearTimeout(this.timeout);
        this.timeout = null;
        this.results = null;

        if (!results || results.length === 0) {
          return;
        }

        // ASSERT: results is defined and at least one result
        this.trace(`results ok: len=${results.length}`);
        this.results = results;
      },
      onQueryError (err) {
        if (err) {
          this.trace(`onQueryResult error: ${err}`);
        }
      }
    },
    watch: {
      // Input string value changed
      search (queryStr) {
        if (this.timeout || this.selecting) {
          // Query or selection in progress
          this.trace(`query or selection in progress...`);
          return;
        }
        if (!queryStr || queryStr.length === 0) {
          // Query reset
          this.trace(`queryStr none`);
          this.results = null;
          return
        }

        // ASSERTION: queryStr is valid
        queryStr = queryStr.trim();

        // Only query if minimal number chars typed and querystring has changed
        queryStr.length >= this.minChars && queryStr !== this.lastQueryStr && this.querySelections(queryStr);
        this.lastQueryStr = queryStr;
      },
      // User has selected entry from suggested items
      selected (item) {
        if (!item.hasOwnProperty('text') || !item.hasOwnProperty('value')) {
          return;
        }
        this.selecting = true;
        this.trace(`selected=${item.text}`);

        // Position Map on result
        const result = item.value;
        const mapProjection = this.map.getView().getProjection();
        const coords = fromLonLat([result.lon, result.lat], mapProjection);

        // Prefer zooming to bounding box if present in result
        if (result.hasOwnProperty('boundingbox')) {
          // Result with bounding box.
          // bbox is in EPSG:4326, needs to be transformed to Map Projection (e.g. EPSG:3758)
          const extent = applyTransform(result.boundingbox, getTransform('EPSG:4326', mapProjection));
          this.map.getView().fit(extent);
        } else {
          // No bbox in result: center on lon/lat from result and zoom in
          this.map.getView().setZoom(this.selectZoom);
        }
        this.map.getView().setCenter(coords);
        this.selecting = false;
      }
    },
    mounted () {
      let config = this.$appConfig.modules['wgu-geocoder'] || {};
      this.debug = config.debug || false;
      this.minChars = config.minChars || 3;
      this.queryDelay = config.queryDelay || 300;
      this.selectZoom = config.selectZoom || 16;
      this.placeHolder = config.placeHolder || 'Search for an address';

      // Setup GeocoderController to which we delegate Provider and query-handling
      this.geocoderController = new GeocoderController(config.provider || 'osm', config.providerOptions || {}, this)
    }
  }
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style>
</style>
