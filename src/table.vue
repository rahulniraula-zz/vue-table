<script>
import axios from "axios";
import { util } from "../index.js";
import Paginate from "vuejs-paginate";
export default {
  components: { Paginate },
  data: function () {
    return {
      sortOrder: -1,
      sortColumn: null,
      internalItems: [], //internal representation of items passed as prop or calculated from url
      dataFromServer: {},
      currentPage: 1,
      query: "", // query parameter for server side searching
      localQuery: {},
      externalFilter: {},
    };
  },
  props: {
    params: {
      type: Object,
      default: function () {
        return {};
      },
    },
    columnSortOrder: Object,
    items: Array, //Items passed as prop for local side data
    headingTransformer: Function, // the transformer function if data in the heading needs some transformation before rendering
    valueTransformer: Function, // the transformer function if data  needs some transformation before rendering
    html: Array, //An array of columns which accept html content TOTO
    additionalColumns: Array, //Additional columns which needs to be appended to the table
    additionalColumnsTransformer: Function, //The function which transfoms values for additional columns
    except: Array, // and array of columns excluded from displaying
    url: String, // the url which is used for fetching data from the server
    paginate: {
      type: Object,
      default: function () {
        return { enable: false };
      },
    }, // Pagiantion option for local data //TODO
    perPage: {
      type: Number,
      default: 10,
    },
    tableClass: {
      type: Object,
      default: () => {
        return {
          table: true,
        };
      },
    },
    rowCallback: {
      type: Function,
    },
  },
  mounted: function () {
    this.externalFilter = {};
    util.event.$off("filter");
    util.event.$on("filter", (obj) => {
      this.externalFilter = {
        ...this.externalFilter,
        ...obj,
        page: this.currentPage,
        q: this.query,
      };
      this.fetchData("");
    });
    this.$root.$emit("vueTableMounted");
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
    serialize(obj) {
      var str = [];
      for (var p in obj)
        if (obj.hasOwnProperty(p)) {
          str.push(encodeURIComponent(p) + "=" + encodeURIComponent(obj[p]));
        }
      return str.join("&");
    },
    /**
     * Search for data in the server in the given url
     */
    performSearch() {
      // let q = "?page=" + this.currentPage + "&q=" + this.query;
      this.fetchData();
    },
    /**
     * Handler for server side pagiation
     */
    paginationClickHandler(page) {
      this.externalFilter = {
        ...this.externalFilter,
        page: this.currentPage,
        q: this.query,
      };
      this.fetchData();
    },
    /**
     * Local Search for items received through items props
     * TODO
     */
    performLocalSearch() {
      this.internalItems = this.items.filter((i) => {
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
    fetchData() {
      axios({
        url:
          this.$props.url +
          "?" +
          (Object.keys(this.$props.params).length > 0
            ? this.serialize(this.$props.params) + "&"
            : "") +
          this.serialize({
            ...this.externalFilter,
            q: this.query,
            page: this.currentPage,
          }),
      })
        .then((r) => {
          this.dataFromServer = r.data.data;
          this.dataFromServer ? (this.internalItems = r.data.data.data) : "";
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
      return this.$props.headingTransformer
        ? this.$props.headingTransformer(val)
        : val;
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
            return t[col](obj, obj[col]);
          }
        }
        return obj[col];
      }
    },
  },
  computed: {
    /**
     * Gets an array of columns concatinating the columns of the supplied
     * items and the additional items without the items specified in the except array
     * output=normalColumns + additionalColumns - except
     */
    _cols: function () {
      let output = Array.isArray(this.$props.additionalColumns)
        ? [].concat(this.$props.additionalColumns)
        : [];
      if (this.internalItems && this.internalItems.length > 0) {
        let first_item = this.internalItems[0];
        let headings = Object.keys(first_item);
        if (this.$props.except && Array.isArray(this.$props.except)) {
          headings = headings.filter((h) => {
            return this.$props.except.indexOf(h) === -1;
          });
        }
        let a = [...headings, ...output];
        return a.sort((a, b) => {
          if (this.$props.columnSortOrder) {
            let index = this.$props.columnSortOrder[a]
              ? this.$props.columnSortOrder[a]
              : a.length;
            let index1 = this.$props.columnSortOrder[b]
              ? this.$props.columnSortOrder[b]
              : a.length;
            return index - index1;
          }
          return 1;
        });
      }
      return output;
    },
    /**
     * Returns an array of columns applying headingTransformer function to each heading
     */
    columns: function () {
      if (typeof this.$props.headingTransformer === "function") {
        return this._cols.map((h) => {
          return this.$props.headingTransformer(h);
        });
      }
      return this._cols;
    },
    /**
     * up and down arrows shown on the heading on each click on the heading
     */
    sortClass: function () {
      return {
        "fa fa-arrow-down": this.sortOrder == -1,
        "fa fa-arrow-up": this.sortOrder == 1,
      };
    },
    /**
     * Total page count items used for pagination
     */
    pageCount: function () {
      return Math.ceil(this.internalItems.length / this.$props.perPage);
    },
  },
  watch: {
    /**
     * Watch for changes in items and assign them to local varialble
     */
    items: function (n) {
      this.internalItems = n;
    },
    /**
     * For column wise local search
     * TODO
     */
    _cols: function (n) {
      n.forEach((c) => {
        this.localQuery[c] = "";
      });
    },
  },
  render(h) {
    return (
      <div>
        {this.$props.url ? (
          <div class="btn-group float-right" style="margin-bottom:5px">
            <input
              type="text"
              class="float-right form-control"
              style="margin:0px"
              placeholder="Search"
              v-model={this.query}
              vOn:keyup_enter={this.performSearch.bind(this)}
            />
            <span
              class="fa fa-search btn btn-success"
              style="cursor:pointer"
              vOn:click={this.performSearch.bind(this)}
            />
          </div>
        ) : (
          <div class="btn-group float-right" style="margin-bottom:5px">
            <input
              type="text"
              class="float-right form-control"
              style="margin:0px"
              placeholder="Search"
              v-model={this.query}
              vOn:keyup={this.performLocalSearch.bind(this)}
            />
            <span
              class="fa fa-search btn btn-success"
              style="cursor:pointer"
              vOn:click={this.performLocalSearch.bind(this)}
            />
          </div>
        )}
        <div class="btn-group float-left header-left" style="margin-bottom:5px">
          {this.$scopedSlots.headerLeft
            ? this.$scopedSlots.headerLeft({ ...this.externalFilter })
            : null}
        </div>
        {this.internalItems.length == 0 ? (
          <div class="alert alert-warning text-center" style="clear:both">
            No Data available
          </div>
        ) : (
          <table class={this.$props.tableClass}>
            <thead>
              <tr>
                {this._cols.map((column) => {
                  return (
                    <th
                      key={column}
                      vOn:Click={this.headingClicked.bind(this, column)}
                      style="cursor:pointer"
                      title={"Click To Toggle Sorting Order by " + column}
                    >
                      <span
                        class={this.sortClass}
                        style={{
                          display: this.sortColumn == column ? "" : "none",
                        }}
                      ></span>
                      {this.getHeadingTransformedValue(column)}
                    </th>
                  );
                })}
              </tr>
            </thead>
            <tbody>
              {this.internalItems.map((item, i) => {
                return (
                  <tr
                    key={"item_" + i}
                    class={
                      this.$props.rowCallback &&
                      typeof this.$props.rowCallback === "function"
                        ? this.$props.rowCallback.call(this, item)
                        : []
                    }
                  >
                    {this._cols.map((column, i) => {
                      return (
                        <td key={"column_" + i}>
                          {this.isHtmlValid(item, column)
                            ? this.getValue(item, column).map((i, j) => {
                                if ("comp" in i) {
                                  return <i.comp {...{ props: i.prop }} />;
                                } else {
                                  return (
                                    <span
                                      domPropsInnerHTML={i.item}
                                      vOn:click={i.handler.bind(this, i.type)}
                                      key={"ind_" + j}
                                    ></span>
                                  );
                                }
                              })
                            : this.getValue(item, column)}
                        </td>
                      );
                    })}
                  </tr>
                );
              })}
            </tbody>
          </table>
        )}
        {this.dataFromServer["last_page"] ? (
          <paginate
            class="float-right"
            pageCount={this.dataFromServer["last_page"]}
            containerClass="pagination"
            pageLinkClass="page-link"
            margin-pages={2}
            pageClass="page-item"
            prev-text="Prev"
            next-text="Next"
            prev-link-class="page-link"
            next-link-class="page-link"
            click-handler={this.paginationClickHandler}
            v-model={this.currentPage}
          ></paginate>
        ) : null}
      </div>
    );
  },
};
</script>




