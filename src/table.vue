<template>
  <div>
    <!-- perform server side search if url is passed as prop -->
    <div class="btn-group pull-right" style="margin-bottom:5px" v-if="url">
      <input
        type="text"
        class="pull-right form-control"
        style="margin:0px"
        placeholder="Search"
        v-model="query"
        @keyup.enter="performSearch"
      />
      <span class="fa fa-search btn btn-success" style="cursor:pointer" @click="performSearch"></span>
    </div>
    <!-- end of server side search -->
    <!-- local search enable for less data  -->
    <div class="btn-group pull-right" style="margin-bottom:5px" v-else>
      <input
        type="text"
        class="pull-right form-control"
        style="margin:0px"
        placeholder="Search"
        v-model="query"
        @keyup="performLocalSearch"
      />
      <span class="fa fa-search btn btn-success" style="cursor:pointer" @click="performLocalSearch"></span>
    </div>
    <!-- End of local side search UI -->
    <table class="table table-bordered">
      <!-- Enable column wise search
        Currently done for local data
        TODO
      -->
      <tr v-if="false">
        <td v-for="column in _cols" :key="column">
          <input
            type="text"
            class="form-control"
            v-if="Object.keys(localQuery).length>0"
            :placeholder="'Search By '+column"
            v-model="localQuery[column]"
            @keyup="performLocalSearch"
          />
        </td>
      </tr>
      <!-- Column wise search end portion
      TODO-->

      <!-- Column Heading portion
      Contains click handler as well which sorts the data-->
      <tr>
        <th
          v-for="column in _cols"
          :key="column"
          @click="headingClicked(column)"
          style="cursor:pointer"
          :title="'Click To Toggle Sorting Order by '+column"
        >
          <span :class="sortClass" :style="{display:sortColumn==column?'':'none'}"></span>
          {{getHeadingTransformedValue(column)}}
        </th>
      </tr>
      <tr v-for="(item,i) in internalItems" :key="'item_'+i">
        <td v-for="(column,i) in _cols" :key="'column_'+i">
          <template v-if="isHtmlValid(item,column)">
            <span
              v-for="(i,j) in getValue(item,column)"
              v-html="i.item"
              @click="i.handler(i.type)"
              :key="'ind_'+j"
            ></span>
          </template>
          <template v-else>
            <span>{{getValue(item,column)}}</span>
          </template>
        </td>
      </tr>
    </table>

    <!-- Pagination enabled if url is supplied
    as prop to the component-->
    <paginate
      v-if="dataFromServer['last_page']"
      class="pull-right"
      :pageCount="dataFromServer['last_page']"
      :containerClass="'pagination'"
      :pageLinkClass="'page-link'"
      :margin-pages="2"
      :pageClass="'page-item'"
      :prev-text="'Prev'"
      :next-text="'Next'"
      prev-link-class="page-link"
      next-link-class="page-link"
      :click-handler="paginationClickHandler"
      v-model="currentPage"
    ></paginate>
    <!-- End of server side pagination code  -->

    <!-- Local pagination start -->
    <paginate
      v-if="paginate.enable"
      :pageLinkClass="'page-link'"
      class="pull-right"
      :containerClass="'pagination'"
      :pageClass="'page-item'"
      v-model="currentPage"
      :pageCount="pageCount"
      prev-link-class="page-link"
      next-link-class="page-link"
      :click-handler="localPaginationClickHandler"
    ></paginate>
    <!-- end of local pagination  -->
  </div>
</template>
<script>
import Paginate from "vuejs-paginate";
import axios from "axios";
export default {
  components: { Paginate },
  data: function() {
    return {
      sortOrder: -1,
      sortColumn: null,
      internalItems: [], //internal representation of items passed as prop or calculated from url
      dataFromServer: {},
      currentPage: 1,
      query: "", // query parameter for server side searching
      localQuery: {}
    };
  },
  props: {
    items: Array, //Items passed as prop for local side data
    headingTransformer: Function, // the transformer function if data in the heading needs some transformation before rendering
    valueTransformer: Function, // the transformer function if data  needs some transformation before rendering
    html: Array, //An array of columns which accept html content TOTO
    additionalColumns: Array, //Additional columns which needs to be appended to the table
    additionalColumnsTransformer: Function, //The function which transfoms values for additional columns
    except: Array, // and array of columns excluded from displaying
    url: String, // the url which is used for fetching data from the server
    paginate: {
      type:Object,
      default:function(){
        return {enable:false}
      }
    }, // Pagiantion option for local data //TODO
    perPage: {
      type: Number,
      default: 10
    }
  },
  mounted: function() {
    /**
     * if url is supplied then fetch the data from the url
     * else
     * set the data received in props and set them to internalItems local variable
     */
    if (this.$props.url) {
      let q = "?page=" + this.currentPage;
      this.fetchData(q);
    } else {
      this.$set(this.$data, "internalItems", this.$props.items);
    }
  },
  methods: {
    /**
     * Search for data in the server in the given url
     */
    performSearch() {
      let q = "?page=" + this.currentPage + "&q=" + this.query;
      this.fetchData(q);
    },
    /**
     * Handler for server side pagiation
     */
    paginationClickHandler(page) {
      console.log(this.currentPage);
      let q = "?page=" + this.currentPage;
      this.fetchData(q);
    },
    /**
     * Local Search for items received through items props
     * TODO
     */
    performLocalSearch() {
      this.internalItems = this.items.filter(i => {
        return i.name.indexOf(this.query) != -1;
      });
    },
    /**
     * Local pagiantion Handler
     * TOTO
     */
    localPaginationClickHandler(page) {
      this.currentPage = page;
      this.internalItems = this.$props.items.slice(
        page * this.$props.perPage,
        20
      );
    },
    /**
     * Fetch the data from the server and set it to internalItems
     */
    fetchData(query) {
      axios({
        url: this.$props.url + query
      })
        .then(r => {
          this.dataFromServer = r.data.data;
          this.internalItems = r.data.data.data;
        })
        .catch();
    },
    /**
     * Sort the data in table in either ascending or
     * descending order when column heading is clicked
     */
    headingClicked(col) {
      this.sortOrder = this.sortOrder * -1;
      this.sortColumn = col;
      this.internalItems.sort((a, b) => {
        let vala = a[col] + "".toUpperCase();
        let valb = b[col] + "".toUpperCase();
        if (vala < valb) {
          return -1 * this.sortOrder;
        } else if (valb < vala) {
          return 1 * this.sortOrder;
        }
        return 0 * this.sortOrder;
      });
    },
    /**
     * Transforms the heading value before displaying it
     */
    getHeadingTransformedValue(val) {
      return this.$props.headingTransformer?this.$props.headingTransformer(val):val;
    },
    /**
     * Check wether html listing of column is enabled
     * TODO
     * Now returns value as it is.
     */
    isHtmlValid(item, column) {
      if (this.$props.html) {
        return this.$props.html.indexOf(column) !== -1 ? true : false;
        //TODO selectively enable and disable html escaping according to `html` props value
      }
      return false;
    },
    /**
     * Gets the value for the column optionallly
     * filtering them by transformers supplied as props
     * @returns transformed value applying the tarnsformers
     */
    getValue(obj, col) {
      if (
        Array.isArray(this.$props.additionalColumns) &&
        this.$props.additionalColumns.indexOf(col) !== -1
      ) {
        if (typeof this.$props.additionalColumnsTransformer === "function") {
          let t = this.$props.additionalColumnsTransformer();
          if (col in t) {
            return t[col](obj, col);
          }
        }
      } else {
        if (typeof this.$props.valueTransformer === "function") {
          let t = this.$props.valueTransformer();
          if (col in t) {
            return t[col](obj[col]);
          }
        }
        return obj[col];
      }
    }
  },
  computed: {
    /**
     * Gets an array of columns concatinating the columns of the supplied
     * items and the additional items without the items specified in the except array
     * output=normalColumns + additionalColumns - except
     */
    _cols: function() {
      let output = Array.isArray(this.$props.additionalColumns)
        ? [].concat(this.$props.additionalColumns)
        : [];
      if (this.internalItems.length > 0) {
        let first_item = this.internalItems[0];
        let headings = Object.keys(first_item);
        if (this.$props.except && Array.isArray(this.$props.except)) {
          headings = headings.filter(h => {
            return this.$props.except.indexOf(h) === -1;
          });
        }
        return [...headings, ...output];
      }
      return output;
    },
    /**
     * Returns an array of columns applying headingTransformer function to each heading
     */
    columns: function() {
      if (typeof this.$props.headingTransformer === "function") {
        return this._cols.map(h => {
          return this.$props.headingTransformer(h);
        });
      }
      return this._cols;
    },
    /**
     * up and down arrows shown on the heading on each click on the heading
     */
    sortClass: function() {
      return {
        "fa fa-arrow-down": this.sortOrder == -1,
        "fa fa-arrow-up": this.sortOrder == 1
      };
    },
    /**
     * Total page count items used for pagination
     */
    pageCount: function() {
      return Math.ceil(this.internalItems.length / this.$props.perPage);
    }
  },
  watch: {
    /**
     * Watch for changes in items and assign them to local varialble
     */
    items: function(n) {
      this.internalItems = n;
    },
    /**
     * For column wise local search
     * TODO
     */
    _cols: function(n) {
      n.forEach(c => {
        this.localQuery[c] = "";
      });
    }
  }
};
</script>
<style lang="css">
</style>
