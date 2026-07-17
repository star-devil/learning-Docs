```ts
/** ECharts默认配置 */

const defaultOptions = ref({
	tooltip: {
		position: 'top',
		formatter: function (params: any) {
		// params 中可以自定义多个值
			const [_xIndex, _yIndex, assessmentValue, featureCount, patientId] =
			params.data;
			return `xxx`;
		}
	},
	grid: {
		height: '70%',
		top: '5%',
		left: '10%'
	},
	xAxis: {
		type: 'category',
		data: [],
		splitArea: {
			show: true
		},
		axisLabel: {
			rotate: 45,
			interval: 0,
			fontSize: 11,
			margin: 10,
			fontFamily: 'inherit',
			fontWeight: 500
		},
		axisTick: {
			show: false
		}
	},
	yAxis: {
		type: 'category',
		data: [],
		splitArea: {
			show: true
		},
		axisLabel: {
			fontSize: 11,
			fontFamily: 'inherit',
			fontWeight: 500,
			margin: 8
		},
		axisTick: {
			show: false
		}
	},
	visualMap: {
		min: 0,
		max: 1,
		calculable: false,
		orient: 'horizontal',
		left: 'center',
		bottom: '0%',
		textStyle: {
		fontSize: 11,
		fontFamily: 'inherit'
		},
		realtime: false,
		inRange: {
			color: ['#d9e0f6', '#4884ef']
		},
		dimension: 2, // 这里指定热力图显示的是数组中的第几个维度，默认是最后一个
		pieces: [
			{ value: 0, color: '#d9e0f6', label: '否' },
			{ value: 1, color: '#4884ef', label: '是' }
		]
	},
	series: [
		{
			name: '指标结果',
			type: 'heatmap',
			data: [],
			universalTransition: {
				enabled: true
			},
			label: {
				show: false // 不显示数值标签
			},
			itemStyle: {
				borderColor: 'rgba(0, 0, 0, 0.2)',
				borderWidth: 0.5,
				borderType: 'dashed'
			},
			emphasis: {
				itemStyle: {
				shadowBlur: 10,
				shadowColor: 'rgba(0, 0, 0, 0.5)'
				},
				label: {
					fontWeight: 'bold'
				}
			},
			progressive: 100, // 渐进式渲染优化
			progressiveThreshold: 1 // 启用渐进式渲染
		}
	]
});
```

```ts
// 先初始化
chartInstance.value = init(chartRef.value, null, {
	renderer: 'canvas', // 使用 canvas 渲染提高性能
	useDirtyRect: true // 启用脏矩形优化
});

// 再设置默认配置，最后更新数据
chartInstance.value.setOption(defaultOptions.value);
```

```ts
/**将数据转换为热力图所需的数据格式*/
const transformDataToHeatmap = async (
analysisResults: AnalysisResultData
): Promise<HeatmapData> => {
	if (isEmpty(analysisResults)) {
		return {
			data: []
		};
	}
	const data = Object.entries(analysisResults).flatMap(
	([outterKey, innerObj]) => {
		return Object.entries(innerObj)
		.filter(([key]) => key.startsWith('get_copd_'))
		.map(([_, copdAssessment], innerIndex) => {
			const _copdAssessment = copdAssessment as COPDAssessment;
			const assessmentValue = _copdAssessment.assessment ? 1 : 0;
			const featureCount = Object.keys(_copdAssessment.features).length; 
			return [
				innerIndex,
				patientIndex.value,
				assessmentValue, // visualMap.dimension
				featureCount,
				outterKey
			] as [number, number, number, number, string];
		});
	});

	patientIndex.value++; // y 轴坐标索引
	return {
		data
	};
};
```