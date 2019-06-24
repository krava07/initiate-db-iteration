<template>
    <li class="rule display-flex">
        <template v-if="rule.mode === 'select'">
            <or-select-expression
                :options.sync="fieldsWithMerge"
                :keys="fieldsSelectKeys"
                v-model="fieldComputed"
                class="field flex-1 no-margin"
                hasSearch
                extendableOptions
                :disabled="readonly"
                :steps="steps"
                :step-id="stepId"
            ></or-select-expression>
            <or-select
                :options="operationsComputed"
                v-model="rule.operation"
                :class="['operation', 'no-margin', { error: !v.operation }]"
                placeholder="operator"
                :disabled="readonly"
            ></or-select>
            <or-text-expression
                class="expression flex-1 hide-controls no-margin"
                v-model="rule.expression"
                :readonly="readonly"
                :steps="steps"
                :step-id="stepId"
            ></or-text-expression>
        </template>
        <or-code
            v-else
            class="code-rule flex-1 no-margin"
            v-model="rule.code"
            @input="(value) => $emit('ruleInput', value)"
            :readonly="readonly"
            :steps="steps"
            :step-id="stepId"
            label=""
        ></or-code>
        <span :class="['popover-trigger', { 'code-mode': rule.mode === 'code' }]" ref="popover-trigger">
            <or-icon icon="more_vert" class="grey"></or-icon>
        </span>
        <or-popover
            trigger="popover-trigger"
            open-on="click"
            dropdownPosition="left bottom"
            ref="popover"
        >
            <div class="control" :style="{ display: 'flex', 'align-items': 'center', padding: '6px 0' }">
            <or-button type="secondary" class="flat grey" :disabled="rule.dirty || readonly" @click="$refs.popover.close() || $emit('toggleRuleMode')">
                {{rule.mode === 'code' ? 'UI' : 'Code'}} mode
                <div slot="icon" class="svg-icon-position grey">
                    <svg width="24px" height="14px" :style="{ fill: '#6b7278', transform: 'scale(0.8) translateY(3px)' }">
                        <use xlink:href="#jsIcon"/>
                    </svg>
                </div>
            </or-button>
            </div>
            
            <div class="control" :style="{ display: 'flex', 'align-items': 'center', padding: '6px 0' }">
            <or-button type="secondary" class="flat" @click="$refs.popover.close() || $emit('removeRule')" :disabled="readonly || ruleset.rules.length === 1">
                Delete
                <or-icon
                slot="icon"
                icon="delete_forever"
                :style="{ 'font-size': '20px', 'margin-right': '4px' }"
                ></or-icon>
            </or-button>
            </div>
        </or-popover>
    </li>
</template>

<script>
import * as _ from 'lodash';

export default {
    name: 'rule',
    props: {
        rule: {},
        steps: {},
        stepId: {},
        operations: {},
        readonly: {},
        fields: {},
        // table: {},
        ruleset: {},
        v: {}
    },
    data () {
        const defaultOperators = ['=', '!='];
        const extraOperations = [...defaultOperators, '<', '<=', '>', '>='];
        
        return {
            fieldsSelectKeys: {
                label: 'Label',
                value: 'Name'
            },
            fieldError: false,
            fieldsWithMerge: [...this.expressionizeFields(this.fields)],
            defaultOperators,
            extraOperations
        };
    },
    methods: {
        expressionizeFields (fields) {
            return _.map(fields, (field, fieldName) => ({
                Name: '`' + fieldName + '`',
                Label: fieldName,
                DataType: field.type
            }));
        }
    },
    watch: {
        fieldsComputed () {
            if (/^(`\$\{this.get\('.*'\)\}`)$/.test(this.rule.field)) {
                this.fieldsWithMerge = [_.find(this.fieldsWithMerge, ['Name', this.rule.field]), ...this.fieldsComputed];
            } else {
                this.fieldsWithMerge = [...this.fieldsComputed];
            }
        }
    },
    computed: {
        fieldsComputed () {
            return this.expressionizeFields(this.fields);
        },
        fieldComputed: {
            get () {
                return this.rule.field;
            },
            set (value) {
                const fieldData = _.find(this.fieldsWithMerge, { Name: value });
                let operationsSet = this.extraOperations;
                if (fieldData && fieldData.DataType === 'boolean') {
                    operationsSet = this.defaultOperators;
                }
                if (!_.includes(operationsSet, this.rule.operation)) {
                    this.rule.operation = '';
                }
                this.rule.field = value;
            }
        },
        operationsComputed () {
            return _.filter(this.operations, operation => {
                if (!this.rule.field) {
                    return true;
                }

                const fieldData = _.find(this.fieldsWithMerge, { Name: this.rule.field });

                if (fieldData) {
                    switch (fieldData.DataType) {
                        case 'boolean':
                            return _.includes(this.defaultOperators, operation.value);
                        // case 'date':
                        // case 'double':
                        // case 'long':
                            // return _.includes(this.extraOperations, operation.value);
                        default:
                            return _.includes(this.extraOperations, operation.value);
                    }
                }
                return true;
            });
        }
    }
}
</script>
