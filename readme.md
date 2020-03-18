# Vue JS Datatable component.
* Table Component for Vue js supporting server and client slide rendering with pagination and sorting
* If URL is supplied as a prop, then the data is fetched from the url. Receives data as props too.
# Features
* Server side search
* Local In memory Searching
* Sort the Columns 
* Supports custom heading through custom function.
* Allows transformation of values through custom function.
* Pagination
* Allows selection of columns for display.
* Additional Column can be added on the fly, with custom transformer function

# Props
The component accepts following props for further customization:
```js
items: Array, //Items passed as prop for local side data
    headingTransformer: Function, // the transformer function if data in the heading needs some transformation before rendering
    valueTransformer: Function, // the transformer function if data  needs some transformation before rendering
    html: Array, //An array of columns which accept html content TOTO
    additionalColumns: Array, //Additional columns which needs to be appended to the table
    additionalColumnsTransformer: Function, //The function which transfoms values for additional columns
    except: Array, // and array of columns excluded from displaying
    url: String, // the url which is used for fetching data from the server
    paginate: Object, // Pagiantion option for local data //TODO
    perPage: {
      type: Number,
      default: 10
    }
```
# Usage
import `vue-table` from "geeklearners-vue-table";

register the component 
 ` components: { vTable }`

 ## Minimal Usage

 ```js
 <v-table :items="items">
 
 </v-table>
```
## Advance Usage

```js 
        <v-table
            :headingTransformer="headingTransformer"
            :valueTransformer="valueTransformer"
            :html="['name']" //todo
            :additionalColumns="additionalColumns"
            :additionalColumnsTransformer="additionalColumnsTransformer"
            :except="except" //todo
            :paginate="localPaginate"
            :url="url"
          >
        </v-table>
```
# Example Usage:

