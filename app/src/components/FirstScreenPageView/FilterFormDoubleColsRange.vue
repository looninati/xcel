<template>
	<form @submit.prevent="addFilterHandler" @keyup.stop>
		<table class="table">
			<tbody>
				<tr>
					<td>双列范围逻辑</td>
					<td>
						<span class="select">
							<select v-model="logicOperator">
								<option value="and">且</option>
								<option v-show="!curFilterTagListCount == 0" value="or">或</option>
							</select>
							<p class="val_mask">{{ getLogicOperatorWords(logicOperator) }}</p>
						</span>
					</td>	
					<td>
						<p class="col_placeholder" @click="showColSelectDialog">{{operatorCol.length === 0 ? "请选择列" : formatColGroup}}</p>
					</td>
					<td class="controls">
						<span class="select">
							<select v-model="needConformColIndex">
								<option v-if="needConformColsNum === 0" value="1">
									{{ generateNeedConformColWords(1) }} 
								</option>
								<option v-else v-for="index in needConformColsNum" :key="index"  :value="index">
									{{ generateNeedConformColWords(index) }}
								</option>
							</select>
							<p class="val_mask">{{ generateNeedConformColWords( needConformColIndex ) }}</p>
						</span>
					</td>
					<td>
						<span class="select">
							<select v-model="operator">
								<option v-for="op in filterOptions" 
									:value="op.char">
									{{ op.words }}
								</option>
							</select>
							<p class="val_mask">{{ getOperatorWords(filterOptions, operator) }}</p>
						</span>
					</td>
					<td>
							<input type="text" placeholder="请填写运算符的值" 
								:disabled="operator === 'empty' || operator === 'notEmpty'" 
								v-model="operatorVal">
					</td>
					<td>
						<group-select :group-id="groupId" @changeSelect="changeSelHandler"></group-select>
					</td>
					<td>
						<button type="submit">添加</button>
					</td>
				</tr>
			</tbody>
		</table>
	</form>
</template>

<script>
	import { addFilter, setColSelectDialogStatus, setColSelectType } from '../../vuex/actions'
	import { getActiveSheet, getFilterOptions, getCurFilterTagListCount, getCurColCount } from '../../vuex/getters'
	import { getCharCol, getNumCol, getOperatorWords, getLogicOperatorWords, getFilterWordsPrimitive } from '../../utils/ExcelSet'
	import GroupSelect from './GroupSelect'
	import { ipcRenderer } from 'electron'

	export default {
		components: {
			GroupSelect
		},
		data(){
			return {
				operatorVal: '',
				operatorCol: '', // 最终会转为数组
				operatorColArr: [],
				operator: '>',
				logicOperator: 'and',
				needConformColIndex: 1,
				isConformDoubleCols: true,
				groupId: -1
			}
		},
		vuex: {
			getters: {
				activeSheet: getActiveSheet,
				filterOptions: getFilterOptions,
				curFilterTagListCount: getCurFilterTagListCount,
				curColCount: getCurColCount
			},
			actions: {
				setColSelectDialogStatus,
				setColSelectType,
				addFilter
			}
		},
		mounted() {
			window.eventBus.$on('colSelVal4Double', (colSelectGroup) => {
				this.operatorCol = colSelectGroup
			})
		},
		watch: {
			curFilterTagListCount(){
				if(this.curFilterTagListCount == 0) {
					this.logicOperator = 'and'
				}
			},
			operator() {
				if(this.operator === 'empty' || this.operator === 'notEmpty') {
					this.operatorVal = undefined
				}
			},
			operatorCol() {
				this.generateNeedConformColNum()
			}
		},
		computed: {
			formatColGroup() {
				return this.operatorCol.map((col, index) => {
					return this.getCharCol(col)
				}).join(',')
			},
			needConformColsNum(){
				let operatorColArr = this.operatorColArr
				if(operatorColArr.length >= 2){
					// 取绝对值，让输入的列的顺序无关
					let startIndex = operatorColArr[0],
							endIndex = operatorColArr[operatorColArr.length - 1],
							distance = Math.abs( endIndex - startIndex )
					if(distance === 1) {
						distance = 2
					}else if(distance > 1) {
						distance += 1
					}
					this.needConformColIndex = 1
					return distance 
				}else{
					return 0
				}
			}
		},
		methods: {
			getNumCol,
			getCharCol,
			getLogicOperatorWords,
			getOperatorWords,
			getFilterWordsPrimitive,
			changeSelHandler(groupId) {
				this.groupId = groupId
			},
			showColSelectDialog() {
				this.setColSelectType(2)
				this.setColSelectDialogStatus(true)
			},
			generateNeedConformColWords(index){
				return index !== this.needConformColsNum ? `满足${index}列` : '满足全部列'
			},
			generateNeedConformColNum(){
				let operatorCol = this.operatorCol,
						abs0 = Math.abs(+operatorCol[0]),
						abs1 = Math.abs(+operatorCol[1]),
						startIndex = Math.min(abs0, abs1),
						endIndex = Math.max(abs0, abs1),
						tempColsArr = []

				for(let i = startIndex, len = endIndex; i <= len; i++){
					tempColsArr.push(i - 1)
				}
				this.operatorColArr = tempColsArr
			},
			addFilterHandler() {
				let filterObj = {},
						filterWords = '',
						operatorColArr = this.operatorColArr,
						operator = this.operator,
						operatorWords = this.getOperatorWords(this.filterOptions, operator),
						opVal = this.operatorVal && this.operatorVal.trim(),
						preStr = `第${getCharCol(operatorColArr[0] + 1)}至第${getCharCol(operatorColArr[operatorColArr.length - 1] + 1)}列范围内的值中，至少有${this.needConformColIndex}列`

				if(!this.validateForm({operatorColArr, opVal, operator})) {
					return
				}
				filterWords = preStr + this.getFilterWordsPrimitive({
					operator,
					operatorWords,
					val: opVal
				})
				console.log("this.operatorColArr",  this.operatorColArr)
				console.log("operator", operator)
				filterObj = {
					filterType: 2,
					groupId: this.groupId,
					logicOperator: this.logicOperator,
					col: this.operatorColArr,
					operator: this.operator,
					value: opVal,
					filterWords: filterWords,
					needConformColIndex: this.needConformColIndex
				}
				this.addFilter(filterObj)
				this.operatorVal = ''
				this.operator = '>'
				this.groupId = -1
				this.needConformColIndex = 1
				this.operatorCol = []
			},
			validateForm(args) {
				let { operatorColArr, opVal, operator } = args,
						isValidated = false,
						tipWords = '双列范围逻辑：',
						isNotBelongEmpty = !(operator === 'empty' || operator === 'notEmpty')
				
				if(operatorColArr.length === 0) {
					tipWords += '请填写列'
				}else if(!this.isConformDoubleCols) {
					tipWords += '只能填写两列，它会取指定两列范围内的所有列（包含自身）'
				}else if(operatorColArr[0] + 1 < 1) {
					tipWords += '列从1开始'
				}else if(this.curColCount !== 0 && operatorColArr[operatorColArr.length - 1] + 1 > this.curColCount){
					tipWords += `超过最大列${this.curColCount}`
				}else if(isNotBelongEmpty && opVal.length === 0) {
					tipWords += '请填写运算符的值'
				}else {
					isValidated = true
				}

				if(!isValidated) {
					ipcRenderer.send('sync-alert-dialog', {
						content: tipWords
					})
					return false
				}else {
					 return true
				}
			}
		}
	}
</script>

<style lang="scss" scoped>
	
</style>