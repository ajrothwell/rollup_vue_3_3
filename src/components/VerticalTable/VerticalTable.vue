<template>
  <div v-if="shouldShowTable" class="table-container">
    <h4 v-if="slots.title"
        class="table-title"
    >
      {{ evaluateSlot(slots.title) }}
    </h4>
    <table>
      <tbody>
        <tr v-for="field in slots.fields">
          <th v-html="evaluateSlot(field.label)"
              :style="styles.th || ''"
          />
          <td v-html="evaluateSlot(field.value, field.transforms, nullValue)"
              v-if="hasData"
          />
          <td v-html="''"
              v-if="!hasData"
          />
        </tr>
      </tbody>
    </table>
    <external-link v-if="options && options.externalLink"
                   :options="options.externalLink"
                   :type="'vertical-table'"
    />
  </div>
</template>

<script>
  import TopicComponent from '../TopicComponent.vue';
  import ExternalLink from '../ExternalLink/ExternalLink.vue';

  import jsPDF from 'jspdf';
  import autotable from 'jspdf-autotable';

  export default {
    mixins: [TopicComponent],
    components: {
      ExternalLink,
    },
    computed: {
      styles() {
        if (this.$props.options.styles) {
          return this.$props.options.styles;
        } else {
          return '';
        }
      },
      shouldShowTable() {
        const hasData = this.hasData;
        if (this.item) {
          if (this.item.activeTable) {
            const filterValue = this.item.activeTable;
            const id = this.options.id;
            if (filterValue === id) {
              return true
              // return hasData;
            } else {
              return false;
            }
          } else {
            return true;
            // return hasData;
          }
        } else {
          return true;
          // return hasData;
        }
      },
      hasData() {
        // console.log(this.topicKey, '-', 'hasData?', this.dataSources);
        // console.log('vertTable hasData is running, this.$props.options:', this.$props.options, 'this.$config.dataSources:', this.$config.dataSources);

        // make sure the following is true for all data sources
        if (!this.$props.options.dataSources) {
          return true;
        } else {
          const hasData = this.$props.options.dataSources.every(dataSource => {
            const targetsFn = this.$config.dataSources[dataSource].targets;

            // if the data source is configured for targets
            if (targetsFn) {
              const targetsMap = this.$store.state.sources[dataSource].targets;
              const targets = Object.values(targetsMap);

              // but there are no targets for this address, return false
              if (targets.length === 0) {
                return false;
              }

              // if there are targets for this address, make sure none of them
              // are "waiting"
              return targets.every(target => target.status !== 'waiting');

            // if the data source is not configured for targets, just check that
            // it has data
            } else {
              return !!this.$store.state.sources[dataSource].data;
            }
          });

          return hasData;
        }
      },
    },
    methods: {
      exportTableToPDF() {
        const tableData = []
        let fields = [];
        let totals = {};
        for (let field of this.$props.options.fields) {
          fields.push(field.label)
          totals[field.label] = 0;
        }
        for (let item of this.items) {
          let theArray = []
          for (let field of this.$props.options.fields) {
            if (field['value'](this.$store.state, item) === null) {
              theArray.push('');
            } else {
              theArray.push(field['value'](this.$store.state, item)) || '';
            }

            if (field['value'](this.$store.state, item) === null || isNaN(field['value'](this.$store.state, item))) {
            // if (isNaN(field['value'](this.$store.state, item))) {
              // console.log('isnull:', field['value'](this.$store.state, item));
              totals[field.label] = ''
            } else {
              // console.log('is not null:', field['value'](this.$store.state, item));
              totals[field.label] = totals[field.label] + parseFloat(field['value'](this.$store.state, item));
            }
          }
          tableData.push(theArray);
        }

        if (this.$props.options.totalRow.enabled) {
          let theArray = []
          for (let field of this.$props.options.fields) {
            if (field.label.toLowerCase() === this.$props.options.totalRow.totalField) {
              theArray.push('Total');
            } else if (totals[field.label] === '') {
              theArray.push('');
            } else {
              theArray.push(parseFloat(totals[field.label]).toFixed(2));
            }
          }
          tableData.push(theArray);
        }
        console.log('tableData:', tableData);
        // var doc = new jsPDF();
        var doc = new jsPDF('p', 'pt');
        doc.setFontSize(12);
        let top = 20;
        for (let introLine of this.$props.options.export.introLines) {
          doc.text(10, top, this.evaluateSlot(introLine));
          top = top + 12
        }
        doc.autoTable(fields, tableData, {
          startY: 100,
          tableWidth: 'wrap'
        });

        let filename;
        let fileStart = this.evaluateSlot(this.$props.options.export.file);
        if (fileStart) {
          filename = this.evaluateSlot(this.$props.options.export.file) + '.pdf';
        } else {
          filename = 'export.pdf';
        }
        doc.save(filename);
      },

      exportTableToCSV() {
        // console.log('exportTableToCSV is running');
        const tableData = []

        let fields = [];
        let totals = {};
        for (let field of this.$props.options.fields) {
          fields.push(field.label)
          totals[field.label] = 0;
        }
        for (let item of this.items) {
          let object = {}
          for (let field of this.$props.options.fields) {
            object[field.label] = field['value'](this.$store.state, item);
            if (isNaN(field['value'](this.$store.state, item))) {
              totals[field.label] = null
            } else {
              totals[field.label] = totals[field.label] + parseFloat(field['value'](this.$store.state, item));
            }
          }
          tableData.push(object);
        }

        if (this.$props.options.totalRow.enabled) {
          let object = {}
          for (let field of this.$props.options.fields) {
            if (field.label.toLowerCase() === this.$props.options.totalRow.totalField) {
              object[field.label] = 'Total';
            } else {
              object[field.label] = totals[field.label];
            }
          }
          tableData.push(object);
        }
        const opts = { fields };

        try {
          var result, ctr, keys, columnDelimiter, lineDelimiter, data;

          data = tableData || null;
          if (data == null || !data.length) {
              return null;
          }

          columnDelimiter = ',';
          lineDelimiter = '\n';
          // columnDelimiter = args.columnDelimiter || ',';
          // lineDelimiter = args.lineDelimiter || '\n';

          keys = Object.keys(data[0]);

          result = '';

          for (let introLine of this.$props.options.export.introLines) {
            result += this.evaluateSlot(introLine);
            result += lineDelimiter;
          }

          result += lineDelimiter;
          result += keys.join(columnDelimiter);
          result += lineDelimiter;

          data.forEach(function(item) {
              ctr = 0;
              keys.forEach(function(key) {
                  if (ctr > 0) result += columnDelimiter;

                  result += item[key] || '';
                  ctr++;
              });
              result += lineDelimiter;
          });

          let csv = result;

          data = null;
          let filename;
          let link;

          // filename = 'export.csv';
          let fileStart = this.evaluateSlot(this.$props.options.export.file);
          if (fileStart) {
            filename = this.evaluateSlot(this.$props.options.export.file) + '.csv';
          } else {
            filename = 'export.csv';
          }

          if (!csv.match(/^data:text\/csv/i)) {
              csv = 'data:text/csv;charset=utf-8,' + csv;
          }
          data = encodeURI(csv);

          link = document.createElement('a');
          link.setAttribute('href', data);
          link.setAttribute('download', filename);
          link.click();

        } catch (err) {
          console.error(err);
        }

      },
    }
  };
</script>

<style scoped>
  table {
    margin: 0;
  }

  th, td {
    font-size: 15px;
    text-align: left;
  }

  th {
    width: 30%;
  }

  .external-link {
    padding-top: 5px;
  }

  .table-title {
    /*too much*/
    /*margin-top: 1rem;*/
  }

  .table-container {
    /*this was too much padding for the parcel table, taking out for now*/
    /*padding-top: 1rem;*/
    margin-bottom: 10px !important;
  }
</style>
