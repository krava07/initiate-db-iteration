<template>
    <div>
        <div v-if="!availableTables.length ">
            <p>You don't have any tables to operate with. Please, go to <a :href="schema.uiUrl" target="_blank">Files & Data</a> section and create a table first.</p>
        </div>
        <or-select-expression
            v-else
            v-model="selectedTableModel"
            :keys="tableSelectKeys"
            :options.sync="availableTables"
            :steps="steps" :step-id="stepId"
            hasSearch extendableOptions
            class="justify-label width-100"
            :disabled="readonly"
            :invalid="$v.schema.table.$error"
            @input="value => !value || $v.schema.table.$touch()"
            :error="$v.schema.table.exists ? 'The Table is required.' : 'The previously selected table was removed / or flow was moved to another user account.'"
        >
            Table
            <div>
                <or-button v-if="tableFields.length" color="primary" type="secondary" class="flat low-text" v-on:click.stop.prevent="$refs['fields-modal'].open()">
                    Table fields
                </or-button>
                <or-icon-button title="Refresh tables" type="secondary" class="grey" icon="replay" @click.stop.prevent="getTables" :disabled="readonly"></or-icon-button>
                <a :href="schema.uiUrl" target="_blank" title="Open Custom Data application in new tab">
                    <or-icon icon="open_in_new" type="secondary" class="grey"></or-icon>
                </a>
            </div>
        </or-select-expression>
        <or-modal ref="fields-modal" title="Table Fields" type="medium">
            <div slot="default" v-if="tableFields.length">
                <p style="color:#B1B1B1">Here you can find field labels with the corresponding JSON label and field data type.</p>
                <div class="table-fields">
                    <div class="table-row table-row--head">
                        <div class="table-row__label">Field label</div>
                        <!--<div class="table-row__name">JSON label</div>-->
                        <div class="table-row__type">Data type</div>
                    </div>
                    <div v-for="item in tableFields" :key="item.name" class="table-row">
                        <div class="table-row__label">{{ item.name }}</div>
                        <!--<div class="table-row__name">{{ item.Name }}</div>-->
                        <div class="table-row__type">{{ item.type || 'unknown data type' }}</div>
                    </div>
                </div>
            </div>
        </or-modal>
        <or-modal ref="table-modal" title="Custom data" type="medium" :dismissible="false" removeCloseButton>
            <div class="message">
            All created settings for the current table will be lost. Are you sure you want to continue?
            </div>
            <div slot="footer">
                <or-button color="red" @click="discardChangeTable">No</or-button>
                <or-button color="primary" @click="confirmChangeTable">Yes</or-button>
            </div>
        </or-modal>
        <or-collapsible
            title="Filters to find data"
            open
            borderless
        >
            <condition-builder
                v-model="schema.conditions"
                :table="selectedTable"
                :steps="steps"
                :step-id="stepId"
                :fields="fields"
                :query.sync="schema.query"
                :readonly="readonly"
                :v="$v.schema.conditions.rulesets.$each"
                :allowRemoveLast="true"
            ></condition-builder>
        </or-collapsible>
        <or-collapsible
            title="Iteration and output"
            open
            borderless
        >
            <or-tabs class="flex-box" fullwidth @tab-change="onTabChange" ref="tabs">
                <or-tab title="One by one" id="one-by-one"></or-tab>
                <or-tab title="Simultaneously" id="simultaneously"></or-tab>
            </or-tabs>
            <div class="display-flex">
                <or-text-expression
                    label="Records per page"
                    help-text="Default value: 30"
                    placeholder="Enter a number"
                    v-model="schema.limit"
                    class="no-margin m-r-m flex-1 width-0"
                    :steps="steps"
                    :step-id="stepId"
                    :readonly="readonly"
                    :invalid="$v.schema.limit.$error"
                    error="The Limit must be a valid Javascript code."
                    @input="$v.schema.limit.$touch()"
                ></or-text-expression>
                <div class="display-flex flex-1">
                    <or-select-expression
                        hasSearch
                        extendableOptions
                        label="Sort by column"
                        placeholder="Select a column"
                        v-model="sortField"
                        :options.sync="fieldsForSort"
                        class="flex-1 no-margin m-r-s sort-select width-0"
                        :disabled="readonly"
                        :steps="steps"
                        :step-id="stepId"
                    ></or-select-expression>
                    <or-icon-button
                        :icon="sortDirection === '+' ? 'arrow_upward' : 'arrow_downward'"
                        color="primary"
                        type="secondary"
                        class="flat sort-icon"
                        @click="toggleSort"
                        :disabled="readonly"
                        :tooltip="sortDirection === '+' ? 'Ascending' : 'Descending'"
                    ></or-icon-button>
                </div>
            </div>
            <div class="relative-position output-wrapper">
                <a href="#" @click.prevent="() => readonly || refillOutputExample()" class="link-button back-to-default-btn" v-if="!readonly && outputExampleIsDirty">Back to default</a>
                <or-code
                    v-model="outputExample"
                    label="Output example"
                    :readonly="readonly"
                    :invalid="$v.schema.outputExample.$error"
                    error="The Output example must a valid Javascript code."
                    @input="$v.schema.outputExample.$touch()"
                ></or-code>
            </div>
        </or-collapsible>
    </div>
</template>
<script>
    // import * as _ from 'lodash';
    import {validators} from '_validators';
    import ConditionBuilder from './conditionBuilder.vue';
    // import email from './email.vue';
    // import password from './password.vue';

    const {required, jsExpressionNonEmptyString, generateValidators} = validators;

    export default {
        name       : 'initiate-db-iteration',
        props      : ['template', 'schema', 'step', 'stepId', 'steps', 'readonly', 'isNew'],
        components : {
            'condition-builder': ConditionBuilder
        },

        data () {
            return {
                respTables: [], // backup list to avoid update from or-select
                availableTables: [],
                newTable: null,
                selectedTable: {},
                headers: {
                    'Content-Type': 'application/json',
                    Authorization: ''
                },
                tableSelectKeys: {
                    label: 'Label',
                    value: 'Id'
                },
                fieldsForSort: [{
                    label: '<< No field selected >>',
                    value: ''
                }],
                fields: {}
            };
        },

        created () {},
        mounted() {
            console.log('this', this);
            console.log('NEW!!!!')
            // const currentEndpoint = `${this.$flow.customDataApiUrl}/api/`;
            const currentEndpoint = `https://tablesapi-qa.onereach.ai/${this.$flow.accountId}/${this.$flow.userId}/`;
            if (this.schema.endpoint !== currentEndpoint) {
                this.schema.endpoint = currentEndpoint;
            }
            const uiUrl = `${this.$flow.customDataUrl}/tables/index.html`;
            if (this.schema.uiUrl !== uiUrl) {
                this.schema.uiUrl = uiUrl;
            }
            
            const activeTab = this.isOneByOne ? 'one-by-one' : 'simultaneously';
            this.$nextTick(() => {
                this.$refs.tabs.setActiveTab(activeTab);
            });
            
            if (this.$store && this.$store.state && this.$store.state.auth && this.$store.state.auth.token) {
                this.headers.Authorization = this.$settings.token;
                this.schema.authToken = this.$settings.token;
                this.getTables().then(this.getTableFields);
            } else {
                console.error('Cannot find authorization token', this);
            }
        },

        computed   : {
            outputExample: {
                get () {
                    return _.isNil(this.schema.outputExample) ? JSON.stringify(this.step.outputExample, null, 2) : this.schema.outputExample;
                },
                set (newValue) {
                    try {
                        const parsedValue = new Function(`return ${newValue};`)();
                        // this.$emit('update:outputExample', parsedValue);
                        this.step.outputExample = parsedValue;
                    } catch (e) {}
                    this.schema.outputExample = newValue;
                }
            },
            tableFields () {
                return _.map(this.fields, (value, key) => ({
                    name: key,
                    type: value.type
                }));
            },
            sortTableFields () {
                return this.tableFields.map(field => ({ label: field.name, value: '`' + field.name + '`' }));
            },
            outputExampleIsDirty () {
                return this.output !== this.getDefaultOutputExample();
            },
            selectedTableModel: {
                get () {
                    return this.schema.table;
                },
                set (newValue) {
                    if (this.schema.table && this.schema.table !== newValue) {
                        if (this.isDirty()) {
                            this.newTable = newValue;
                            this.$refs['table-modal'].open();
                        } else {
                            this.resetSchema();
                            this.schema.table = newValue;
                            this.selectedTable = _.find(this.availableTables, { Id: this.schema.table });
                        }
                    } else {
                        this.schema.table = newValue;
                        this.selectedTable = _.find(this.availableTables, { Id: this.schema.table });
                    }
                }
            },
            sortField: {
                get () {
                    return this.schema.sortField;
                },
                set (newValue) {
                    this.schema.sortField = newValue;
                }
            },
            sortDirection: {
                get () {
                    return this.schema.sortDirection;
                },
                set (newValue) {
                    this.schema.sortDirection = newValue;
                }
            },
            isMergeField () {
                return this.selectedTableModel.includes('this.get');
            },
                
            isOneByOne: {
                get () {
                    return this.schema.isOneByOne;
                },
                set (value) {
                    this.schema.isOneByOne = value;
                }
            }
        },

        watch: {
            selectedTableModel (newValue, oldValue) {
                this.getTableFields().then(() => {
                    if (/^(`\$\{this.get\('.*'\)\}`)$/.test(this.sortField)) {
                        this.fieldsForSort = [{
                            label: "<< No field selected >>",
                            value: ''
                        }, ...this.fieldsForSort, ...this.sortTableFields];
                    } else {
                        this.fieldsForSort = [{
                            label: "<< No field selected >>",
                            value: ''
                        }, ...this.sortTableFields];
                    }
                    this.refillOutputExample()
                });
                
                // if (this.isMergeField) {
                //     this.model.tableExists = true;
                // } else {
                //     this.model.tableExists = _.some(this.respTables, { Id: this.model.table });
                // }
            },
            isOneByOne: 'refillOutputExample'
        },

        methods : {
            onTabChange (tabId) {
                if (tabId === 'one-by-one') {
                    this.isOneByOne = true;
                } else if (tabId === 'simultaneously') {
                    this.isOneByOne = false;
                }
            },
            resetSchema () {
                this.schema.conditions = {
                    rulesets: [{
                    title: 'New filter',
                    rules: [{
                        id: libs.uuid.v4(),
                        field: '',
                        operation: '$eq',
                        expression: '``',
                        mode: 'select',
                        code: '',
                        dirty: false
                    }],
                    mode: 'select',
                    code: '',
                    dirty: false
                    }]
                };
                
                this.schema.fields = {};
                this.schema.query = '';
                this.schema.queryObject = true;
                this.schema.outputExample = '';
                this.schema.limit = '`30`';
                this.schema.skip = '``';
                this.sortField = '``';
                this.sortDirection = '+';
            },
            isDirty () {
                try {
                    // return !_.isEmpty(eval(`(${this.model.query})`).$or);
                    return !_.isEmpty(eval(`(${this.schema.query})`).query_string.query);
                } catch (e) {
                    return true;
                }
            },
            discardChangeTable () {
                this.$refs['table-modal'].close();
                this.newTable = null;
            },
            confirmChangeTable () {
                this.$refs['table-modal'].close();
                this.selectedTableModel = this.newTable;
                this.selectedTable = _.find(this.availableTables, { Id: this.selectedTableModel });
                this.resetSchema();
                this.newTable = null;
            },
            getTables () {
                return this.$http.get(`${this.schema.endpoint}indexes-statistics`, {
                    headers: this.headers
                }).then(dataResponse => {
                    if (dataResponse.data && dataResponse.data.indices) {
                        const respTables = Object.keys(dataResponse.data.indices)
                            .map(table => ({
                            Label: table,
                            Id: '`' + table + '`'
                            }));
                        const availableTables = [{
                            Label: "<< No table selected >>",
                            Id: '``'
                        }].concat(respTables);
                        this.respTables = availableTables;
                        const selectedTable = _.find(this.availableTables, { Id: this.selectedTableModel });
                        if (this.isMergeField && selectedTable) {
                            this.availableTables = availableTables.concat(selectedTable);
                        } else {
                            this.availableTables = availableTables;
                        }
                        this.selectedTable = _.find(this.availableTables, { Id: this.selectedTableModel });
                    }
                }, (jq, status, error) => {
                    console.error(`CustomData Tables Fail: ${status}`, jq);
                    if (jq && jq.status == 403) {
                        this.availableTables = [];
                        this.selectedTable = null;
                    }
                });
            },
            getTableFields () {
                const table =  this.selectedTableModel.slice(1, -1);
                if (!table || this.isMergeField) {
                    this.fields = {};
                    return Promise.resolve();
                }
                return this.$http.get(`${this.schema.endpoint}mappings`, {
                    headers: this.headers,
                    params: {
                        index: table
                    }
                }).then(resp => {
                    if (!resp.data) {
                        this.fields = {};
                        return;
                    }
                    this.fields = resp.data;
                });
            },
            toggleSort () {
                this.sortDirection = this.sortDirection === '+' ? '-' : '+';
            },
            groupOutputFields (fields, depth) {
                if (fields.length === 1 && !fields[0].path.length) {
                    switch (fields[0].type) {
                        case 'String':
                        case 'keyword':
                            return '"some string"';
                        case 'Number':
                        case 'long':
                            return '123';
                        case 'Boolean':
                        case 'boolean':
                            return 'true';
                        default:
                            return '"some string"';
                    }
                }

                const spaces = _.range(2 * depth).map(() => ' ').join('');
                const jsonFields = _.chain(fields)
                    .groupBy(field => field.path[0])
                    .toPairs()
                    .map(([parent, group]) => {
                        const arrayField = _.find(group, fd => fd.type === 'Array' && fd.path.length === 1);
                        const subArrayFields =_.reject(group, fd => fd.type === 'Array' && fd.path.length === 1);
                        const fieldsToGroup = arrayField && arrayField.arrayType !== 'Object' ?
                            [{ type: arrayField.arrayType, path: [] }] :
                            _.map(subArrayFields, fd => ({
                                path: fd.path.slice(1),
                                type: fd.type,
                                arrayType: fd.arrayType
                            }));

                        const subObject = groupFields(fieldsToGroup, depth + 1);
                        if (arrayField) {
                        return `${spaces}  "${parent}": [${subObject}]`
                        }
                        return `${spaces}  "${parent}": ${subObject}`;
                    })
                    .join(',\n')
                    .value();
                
                return `{\n${jsonFields}\n${spaces}}`;
            },
            getDefaultOutputExample () {
                let basicFields = [
                    { path: ['_id'], type: 'String' },
                    { path: ['_index'], type: 'String' },
                    { path: ['_score'], type: 'Number' },
                    { path: ['_type'], type: 'String' }
                ];
                
                if (!this.isOneByOne) {
                    basicFields = basicFields.map(fieldConf => {
                        fieldConf.path.unshift('hits');
                        return fieldConf;
                    });
                    basicFields = [
                        {path: ['total'], type: 'Number'},
                        {path: ['max_score'], type: 'Number'},
                        {path: ['hits'], type: 'Array', arrayType: 'Object'}
                    ].concat(basicFields);
                }
                if (this.selectedTableModel !== '``') {
                    const tableFields = this.tableFields.map(field => ({
                        type: field.type,
                        path: this.isOneByOne ? ['_source', ...field.name.split('.')] : ['hits', '_source', ...field.name.split('.')]
                    })).filter(fd => fd.type !== 'Object' && fd.path[1] !== '_MetaData');
                    basicFields = basicFields.concat(tableFields);
                    return this.groupOutputFields(basicFields, 0);
                } else if (this.isMergeField) {
                    return this.groupOutputFields(basicFields, 0);
                } else {
                    return '';
                }
            },
            refillOutputExample () {
                this.outputExample = this.getDefaultOutputExample();
            }
        },

        validations () {
            return {
                schema : validator(this.template)
            };
        }

    };

    export const data = (template) => ({
        // email               : '',
        // emailPlaceholder    : template.emailPlaceholder,
        // emailLabel          : template.emailLabel,
        // password            : '',
        // passwordPlaceholder : template.passwordPlaceholder,
        // passwordLabel       : template.passwordLabel,
        initiateDBIterationStep: true,
        isOneByOne: true,
        fields: {}, // ?
        authToken: '',
        endpoint: '',
        uiUrl: '',
        table: '``',
        query: '',
        conditions: {
            rulesets: [{
                title: 'new rule',
                rules: [{
                    id: 'initial',
                    field: '``',
                    operation: '$eq',
                    expression: '``',
                    mode: 'select',
                    code: '',
                    dirty: false
                }],
                mode: 'select',
                code: '',
                dirty: false
            }]
        },
        limit: '`30`',
        skip: '``',
        sortDirection: '+',
        sortField: '``',
        queryObject: true,
        outputExample: ''
    });

    // export const toJson = (data, key) => {
    //     if (['table', 'limit', 'sortField'].includes(key)) {
    //         return data;
    //     }
    //     if (_.isArray(data)) { 
    //         return `[${_.map(data, toJson).join(',')}]`;
    //     }
    //     if (_.isObject(data)) {
    //         return `{${_.map(data, (value, key) => `${key}: ${toJson(value, key)}`).join(',')}}`;
    //     }
    //     return JSON.stringify(data);
    // };

    export const toJSON = ({schema, inputData, context}) => {
        return ({
        // [inputData.variableName]: JSON.stringify(schema[inputData.variableName])
        initiateDBIterationStep: JSON.stringify(schema.initiateDBIterationStep),
        isOneByOne: JSON.stringify(schema.isOneByOne),
        // fields: {}, // ?
        authToken: JSON.stringify(schema.authToken),
        endpoint: JSON.stringify(schema.endpoint),
        uiUrl: JSON.stringify(schema.uiUrl),
        table: schema.table,
        query: JSON.stringify(schema.query),
        limit: schema.limit,
        skip: schema.skip,
        sortDirection: JSON.stringify(schema.sortDirection),
        sortField: schema.sortField,
        // queryObject: true,
        })
    };

    export const validator = (template) => {
        return {
            // email    : generateValidators(template.validateRequired, {required}),
            // password : generateValidators(template.validateRequired, {required}),
            table: {
                required (table, schema) {
                    return validators.required && table !== '``';
                }
            },
            limit: {
                validJs (limit, schema) {
                    return validators.jsExpression(limit);
                }
            },
            skip: {
                validJs (skip, schema) {
                    return validators.jsExpression(skip);
                }
            },
             outputExample: {
                jsCode: code => !code || validators.jsExpression(code)
            },
            conditions: {
                rulesets: {
                    $each: {
                        hasNoRepeatitions (ruleset) {
                            if (ruleset.mode === 'code') {
                                return true;
                            }
                            
                            const selectModeRules = _.filter(ruleset.rules, ({mode}) => mode === 'select');

                            return selectModeRules.length === _.chain(selectModeRules)
                                .groupBy(({field, operation}) => `${field}-${operation}`)
                                .keys()
                                .value()
                                .length;
                        },
                        rules: {
                            $each: {
                                opearation (rule) {
                                    if (rule.mode === 'code') {
                                        return true;
                                    }
                                    const fieldName = rule.field.slice(1, -1);
                                    const isMergeField = fieldName.includes('this.get');
                                    const fieldObj = this.schema.tableFields.find(field => field.name === fieldName);
                                    if (!isMergeField && fieldObj && 
                                        (
                                            (fieldObj.DataType === 'Number' && ['contains', 'not contain', 'begins with'].includes(rule.operation)) ||
                                            (fieldObj.DataType === 'Boolean' && !['$eq', '$ne'].includes(rule.operation))
                                        )
                                    ) {
                                        return false;
                                    }
                                    return !!rule.operation;
                                }
                            }
                        }
                    }
                }
            }
        };
    };

    export const meta = {
        name    : 'initiate-db-iteration',
        type    : 'onereach-studio-form-editor',
        version : '1.0'
    };
</script>

<style scoped lang="scss" rel="stylesheet/scss">
    @import '../scss/colors.scss';
    @import '../scss/main.scss';
</style>


<style lang="scss" rel="stylesheet/scss">
    @import '../scss/colors.scss';

</style>
