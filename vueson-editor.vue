<template>
  <div>
    <div class="jsoneditor-vue"></div>
  </div>
</template>

<script>
  import './assets/jsoneditor.css'
  import JsonEditor from './assets/jsoneditor.min'
  import _ from 'lodash'

  export default {
    props: ['value', 'schema'],
    data () {
      return {
        editor: null,
        error: false,
        sch: this.schema || {},
        json: this.value || {}
      }
    },
    mounted () {
      var self = this
      var options = {
        mode: 'tree',
        modes: ['tree', 'form'], //, 'code', 'text', 'view'
        search: false,
        onError () {
          self.$emit('json-error', true)
        },
        onChange () {
          try {
            var json = self.editor.get()
            self.error = false
            self.$emit('json-error', false)
          } catch (e) {
            // console.log('e:', e)
            self.error = true
            self.$emit('json-error', true)
          }
          if (!self.error) {
            self.json = json
            self.$emit('json-change', json)
          }
        }
      }
      this.editor = new JsonEditor(this.$el.querySelector('.jsoneditor-vue'), options, Object.assign({}, this.schemaToJSON(this.sch), this.json))
      this.editor.setSchema(this.sch)
    },
    methods: {
        schemaToJSON (schema, models, modelsToIgnore, modelPropertyMacro) {
          var that = this
          var simpleRef = function (name) {
            if (typeof name === 'undefined') {
              return null;
            }

            if (name.indexOf('#/definitions/') === 0) {
              return name.substring('#/definitions/'.length);
            } else {
              return name;
            }
          };

          var resolveSchema = function (schema) {
            if (_.isPlainObject(schema.schema)) {
              schema = resolveSchema(schema.schema);
            }

            return schema;
          };

          // Resolve the schema (Handle nested schemas)
          schema = resolveSchema(schema);

          if(typeof modelPropertyMacro !== 'function') {
            modelPropertyMacro = function(prop){
              return (prop || {}).default;
            };
          }

          modelsToIgnore= modelsToIgnore || {};

          var type = schema.type || 'object';
          var format = schema.format;
          var model;
          var output;

          if (!_.isUndefined(schema.example)) {
            output = schema.example;
          } else if (_.isUndefined(schema.items) && _.isArray(schema.enum)) {
            output = schema.enum[0];
          }

          if (_.isUndefined(output)) {
            if (schema.$ref) {
              model = models[simpleRef(schema.$ref)];

              if (!_.isUndefined(model)) {
                if (_.isUndefined(modelsToIgnore[model.name])) {
                  modelsToIgnore[model.name] = model;
                  output = that.schemaToJSON(model.definition, models, modelsToIgnore, modelPropertyMacro);
                  delete modelsToIgnore[model.name];
                } else {
                  if (model.type === 'array') {
                    output = [];
                  } else {
                    output = {};
                  }
                }
              }
            } else if (!_.isUndefined(schema.default)) {
              output = schema.default;
            } else if (type === 'string') {
              if (format === 'date-time') {
                output = new Date().toISOString();
              } else if (format === 'date') {
                output = new Date().toISOString().split('T')[0];
              } else {
                output = 'string';
              }
            } else if (type === 'integer') {
              output = 0;
            } else if (type === 'number') {
              output = 0.0;
            } else if (type === 'boolean') {
              output = true;
            } else if (type === 'object') {
              output = {};

              _.forEach(schema.properties, function (property, name) {
                var cProperty = _.cloneDeep(property);

                // Allow macro to set the default value
                cProperty.default = modelPropertyMacro(property);

                output[name] = that.schemaToJSON(cProperty, models, modelsToIgnore, modelPropertyMacro);
              });
            } else if (type === 'array') {
              output = [];

              if (_.isArray(schema.items)) {
                _.forEach(schema.items, function (item) {
                  output.push(that.schemaToJSON(item, models, modelsToIgnore, modelPropertyMacro));
                });
              } else if (_.isPlainObject(schema.items)) {
                output.push(that.schemaToJSON(schema.items, models, modelsToIgnore, modelPropertyMacro));
              } else if (_.isUndefined(schema.items)) {
                output.push({});
              } else {
                console.log('Array type\'s \'items\' property is not an array or an object, cannot process');
              }
            }
          }
          return output;
        }
    },
    watch: {
      value: function () {
        this.json = this.value
        if(this.editor) this.editor.set( Object.assign({}, this.schemaToJSON(this.sch), this.json))
      },
      schema: function () {
        this.sch = this.schema
        if(this.editor) this.editor.setSchema(this.sch)
        if(this.editor) this.editor.set( Object.assign({}, this.schemaToJSON(this.sch), this.json))
      }
    }
  }
</script>

<style>
  .ace_line_group {
    text-align: left;
  }

  .json-editor-container {
    display: flex;
    width: 100%;
  }

  .json-editor-container .tree-mode {
    width: 50%;
  }

  .json-editor-container .code-mode {
    flex-grow: 1;
  }

  .jsoneditor-btns{
    text-align: center;
    margin-top:10px;
  }

  .jsoneditor-vue .jsoneditor-outer{
    min-height:150px;
  }

  .jsoneditor-vue div.jsoneditor-tree{
    min-height: 350px;
  }

  .json-save-btn{
    background-color: #20A0FF;
    border: none;
    color:#fff;
    padding:5px 10px;
    border-radius: 5px;
  }

  .json-save-btn:focus{
    outline: none;
  }

  .json-save-btn[disabled]{
    background-color: #1D8CE0;
  }

  code {
    background-color: #f5f5f5;
  }
</style>
