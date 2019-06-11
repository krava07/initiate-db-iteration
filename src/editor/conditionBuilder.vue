<template>
    <div>
        <ul class="ruleset-list">
            <div class="placeholder-text" v-if="!value.rulesets.length">No query specified. All records will be returned as a result</div>
            <template v-for="(ruleset, index) in value.rulesets">
                <li :key="index" :class="['ruleset-item', { invalid: v[index].$error }]">
                    <div class="header display-flex">
                    <or-textbox
                        class="inline-input flex-1"
                        v-model="ruleset.title"
                        :disabled="readonly"
                    ></or-textbox>
                    <div class="display-flex align-items-center m-b-6">
                        <or-icon-button title="Duplicate" type="secondary" class="grey medium" icon="content_copy" @click="onDuplicate(ruleset)" :disabled="readonly"></or-icon-button>
                        <or-icon-button title="Toggle mode" type="secondary" class="grey" :disabled="ruleset.dirty || readonly" @click="toggleRulesetMode(ruleset)">
                            <div slot="default" class="svg-icon-position">
                                <svg width="24px" height="14px">
                                    <use xlink:href="#jsIcon"/>
                                </svg>
                            </div>
                        </or-icon-button>
                        <or-icon-button title="Delete" type="secondary" class="grey medium" icon="delete_forever" @click="onRemoveRuleset(ruleset)" :disabled="(value.rulesets.length < 2 && !allowRemoveLast) || readonly"></or-icon-button>
                    </div>
                    </div>
                    <template v-if="ruleset.mode === 'select'">
                        <ul class="ruleset">
                            <rule
                                v-for="(rule, ruleIndex) in ruleset.rules"
                                :key="rule.id"
                                :rule="rule"
                                :steps="steps"
                                :step-id="stepId"
                                :operations="operations"
                                :readonly="readonly"
                                :fields="fields"
                                :table="table"
                                :ruleset="ruleset"
                                :v="v[index].rules.$each[ruleIndex]"
                                @ruleInput="(value) => onRuleInput(rule, value)"
                                @toggleRuleMode="toggleRuleMode(ruleset, rule)"
                                @removeRule="onRemoveRule(ruleset, rule)"
                            ></rule>
                        </ul>
                        <a href="#" @click.prevent="() => readonly || addRule(ruleset)" class="link-button" v-if="!readonly"><or-icon icon="add"></or-icon>add new rule</a>
                        <!-- <div v-if="v[index].$error && !v[index].hasIndexed" class="error">At least one indexed field should be included into the query.</div> -->
                        <div v-if="v[index].$error && !v[index].hasNoRepeatitions" class="error">The query cannot contain rules with equal fields and operations.</div>
                        <div v-if="v[index].$error && v[index].hasIndexed && v[index].hasNoRepeatitions" class="error">The query cannot contain rules with empty operations.</div>
                    </template>
                    <or-code
                        v-else
                        class="code-ruleset"
                        v-model="ruleset.code"
                        @input="(value) => onRulesetInput(ruleset, value)"
                        :readonly="readonly"
                        :steps="steps"
                        :step-id="stepId"
                        label=""
                    ></or-code>
                </li>
                <div :key="index" v-if="index !== value.rulesets.length - 1" class="separator">
                    <div>
                        <span>or</span>
                    </div>
                </div>
            </template>
        </ul>
        <a href="#" @click.prevent="() => readonly || addRuleset()" class="link-button" v-if="!readonly"><or-icon icon="add"></or-icon>add new filter</a>
    </div>
</template>

<script>
// import * as _ from 'lodash';
import rule from './rule.vue';

export default {
    name: 'condition-builder',
    components: {
        rule
    },
    props: {
        value: {
            type: Object,
            default () {
                return {};
            }
        },
        fields: {
            type: Object,
            default () {
                return {};
            }
        },
        table: {
            type: Object,
            default: () => null
        },
        readonly: {
            type: Boolean,
            default: false
        },
        v: {
            type: Object,
            default: () => ({})
        },
        query: String,
        // query: {
        //   type: Object,
        //   default: () => ({})
        // },
        steps: {},
        stepId: {},
        useDataPrefix: {
            type: Boolean,
            default: false
        },
        allowRemoveLast: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            operations: [{
                label: 'equal',
                value: '='
            }, {
                label: '<=',
                value: '<='
            }, {
                label: '>=',
                value: '>='
            }, {
                label: '<',
                value: '<'
            }, {
                label: '>',
                value: '>'
            }, {
                label: 'not equal',
                value: '!='
            }]
            // {
            //   label: 'contains',
            //   value: 'contains',
            //   complex: true,
            //   getRule: expression => `$regex: \`\.*\${${expression}\}\.*\`\, $options: 'i'`
            // },
            // {
            //   label: 'not contain',
            //   value: 'not contain',
            //   complex: true,
            //   getRule: expression => `$regex: \`^((?!\${${expression}})\.)*$\`\, $options: 'i'`
            // },
            // {
            //   label: 'begins with',
            //   value: 'begins with',
            //   complex: true,
            //   getRule: expression => `$regex: \`^\${${expression}\}\`\, $options: 'i'`
            // }]
        };
    },
    watch: {
        value: {
            handler() {
                const orExpressionsList = this.value.rulesets.map(this.rulesetToCode).filter(rs => !!rs);

                // this.$emit('update:query', `{ $or: [${orExpressions}] }`);

                let query = '';
                if (orExpressionsList.length > 1) {
                    query = '(' + orExpressionsList.join(') OR (') + ')'
                }

                const fullQuery = '{ query_string: { query:' + '`' + query + '`' + '} }';
                console.log('query:', fullQuery)
                this.$emit('update:query', fullQuery);
            },
            deep: true
        }
    },
    methods: {
        getEmptyRule() {
            return {
                id: libs.uuid.v4(),
                field: '',
                operation: '=',
                expression: '``',
                mode: 'select',
                code: '',
                dirty: false
            };
        },
        addRule(ruleset) {
            ruleset.rules = [...ruleset.rules, this.getEmptyRule()];
        },
        addRuleset(clone) {
            this.value.rulesets = [...this.value.rulesets, clone || {
                title: 'New filter',
                rules: [this.getEmptyRule()],
                mode: 'select',
                code: '',
                dirty: false
            }];
        },

        onDuplicate(ruleset) {
            this.addRuleset(_.cloneDeep(ruleset));
        },
        toggleRulesetMode(ruleset) {
            ruleset.mode = ruleset.mode === 'code' ? 'select' : 'code';

            if (ruleset.mode === 'code') {
                ruleset.code = this.rulesetToCode(ruleset, 'select');
            }
        },
        toggleRuleMode(ruleset, rule) {
            rule.mode = rule.mode === 'code' ? 'select' : 'code';

            if (rule.mode === 'code') {
                rule.code = this.ruleToCode([rule], rule.field, 'select');
            }
        },
        onRuleInput(rule, value) {
            if (Object.keys(this.fields).length) {
                rule.dirty = this.ruleToCode([rule], rule.field, 'select') !== value;
            }
        },
        onRulesetInput(ruleset, value) {
            if (Object.keys(this.fields).length) {
                ruleset.dirty = this.rulesetToCode(ruleset, 'select') !== value;
            }
        },
        // oldChangeRule (rule) {
        //   const { operation, expression } = rule;
        //   if (!operation || !expression) {
        //     return '';
        //   }

        //   const operationDescriptor = _.find(this.operations, { value: operation });
        //   const toMatch = field.match(/(^`(.*)`$)|(^'(.*)'$)|(^"(.*)"$)/i).filter(e => !!e);
        //   const entry = toMatch[toMatch.length - 1];
        //   const fieldData = _.find(this.fields, { Name: entry });

        //   let parsedExpression = '';
        //   if (fieldData) {
        //     switch (fieldData.DataType) {
        //       case 'Number':
        //         parsedExpression = `parseFloat(${expression})`;
        //         break;
        //       case 'Boolean':
        //         parsedExpression = `${expression} === 'true'`;
        //         break;
        //       case 'String':
        //         parsedExpression = expression;
        //         break;
        //       default:
        //         parsedExpression = expression;
        //     }
        //   } else {
        //     parsedExpression = expression;
        //   }

        //   return operationDescriptor.complex ?
        //     operationDescriptor.getRule(parsedExpression) :
        //     `${operationDescriptor.value}: ${parsedExpression}`;
        // },
        changeRule(fieldName, operator, value) {
            fieldName = fieldName.slice(1, -1);
            value = value.slice(1, -1);
            const notEquals = operator === '!=';
            const type = _.get(this.fields, `${fieldName}.type`);
            if (['!=', '='].includes(operator)) operator = '';
            if (type === 'keyword') {
                value = `"${value}"`;
            }
            let rule = `${fieldName} : ${operator}${value}`;
            if (notEquals) {
                rule = `!(${rule})`;
            }
            return rule;
        },
        ruleToCode(rules, field, source) {
            if ((source || rules[0].mode) === 'code') {
                return rules[0].code;
            }
            if (field && validators.jsExpressionNonEmptyString(field)) {
                // const operations = _.map(rules, this.oldChangeRule).filter(operation => !!operation).join(', ');
                const operations = _.map(rules, rule => this.changeRule(rule.field, rule.operation, rule.expression)).join(' AND ');

                if (!operations) {
                    return 'invalid js';
                }
                // return `[\`${this.useDataPrefix ? 'Data.' : ''}$\{${field}\}\`]: { ${operations} }`;
                // return '(' + operations + ')';
                return operations;
            }

            return '';
        },
        rulesetToCode(ruleset, source) {
            const {
                mode,
                code
            } = ruleset;

            if ((source || mode) === 'code') {
                return code;
            }

            const codeModeRules = _.filter(ruleset.rules, ({
                mode
            }) => mode === 'code');
            const selectModeRules = _.filter(ruleset.rules, ({
                mode
            }) => mode === 'select');
            const groupedRules = _.groupBy(selectModeRules, ({
                field
            }) => field);

            const rulesFields = [
                ..._.map(groupedRules, (rules, field) => this.ruleToCode(rules, field)),
                ..._.map(codeModeRules, rule => this.ruleToCode([rule]))
            ].filter(rule => !!rule).join(' AND ');

            // if (rulesFields) {
            //   return `{\n${rulesFields}\n}`;
            // }

            return rulesFields || '';
        },
        onRemoveRule(ruleset, rule) {
            if (ruleset.rules.length < 2) {
                return;
            }
            ruleset.rules = _.reject(ruleset.rules, currRule => currRule === rule);
        },
        onRemoveRuleset(ruleset) {
            if (this.value.rulesets.length < 2 && !this.allowRemoveLast) {
                return;
            }
            this.value.rulesets = _.reject(this.value.rulesets, currRuleset => currRuleset === ruleset);
        }
    }
}
</script>
