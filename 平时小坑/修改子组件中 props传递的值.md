在 Vue 3 中，props 是只读的，不能直接修改 props 传进来的变量。如果想在组件内部修改 state，有三种常见做法：
1. 使用 defineModel（Vue 3.4+）可以直接实现 props 的双向绑定，无需手动 watch 和 emit。
2. 使用一个本地的副本（如 localState），在组件内部修改它，并通过 emit 通知父组件更新 state。
3. 使用 v-model:state 语法（即 modelValue + update:modelValue），让父组件控制 state 的更新。

## 第一种做法：
```ts
// 子组件中变量 state
const state = defineModel<FieldValues>('state');
// 子组件中绑定 state
<PlusForm v-model="state" ... />

// 父组件中
<ReForm v-model:state="formState" />
```

## 第二种做法：
```ts
// 定义 state
const props = defineProps<{
	state?: FieldValues;
}>();

const emit = defineEmits<{
	(e: "update:state", value: FieldValues): void;
}>();

// 本地副本
const localState = ref(props.state ? { ...props.state } : {});

// 监听 props.state 变化，同步到本地副本
watch(
	() => props.state,
	(val) => {
		if (val) localState.value = { ...val };
	},
	{ deep: true }
);

// 监听本地副本变化，通知父组件
watch(
	localState,
	(val) => {
		emit("update:state", val);
	},
	{ deep: true }
);
```
## 第三种做法：
```ts
// 子组件
<script setup lang="ts">
import { computed } from "vue";
const props = defineProps<{
	state?: any
}>();
const emit = defineEmits<{
	(e: "update:state", value: any): void;
}>();

// 计算属性实现双向绑定
const innerState = computed({
	get: () => props.state,
	set: (val) => emit("update:state", val)
});
</script>

<template>
	<PlusForm v-model="innerState" />
</template>

// 父组件
<template>
	<ReForm v-model:state="formState" />
</template>
<script setup lang="ts">
import { ref } from "vue";
import ReForm from "@/components/ReForm/index.vue";
const formState = ref({});
</script>
```