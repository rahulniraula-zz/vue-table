# Vue JS Datatable component.
* Table Component for Vue js supporting server and client slide rendering with pagination and sorting
* If URL is supplied as a prop, then the data is fetched from the url. Receives data as props too.

# Usage
import `vue-table` from "geeklearners-vue-table";
register the component 
 ` components: { vTable }`

 ## Minimal Usage

 ```
 <v-table :items="items">
 
 </v-table>
```
## Advance Usage

``` 
        <v-table
            :headingTransformer="headingTransformer"
            :valueTransformer="valueTransformer"
            :html="['name']"
            :additionalColumns="additionalColumns"
            :additionalColumnsTransformer="additionalColumnsTransformer"
            :except="except"
            :paginate="localPaginate"
            :url="url"
          >
        </v-table>


# Documentation is not complete. 