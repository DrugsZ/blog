##																									React-Form梳理

#### Form组件

​	`Antd`提供了`Form`组件组件来进行复杂表单的处理,

> 使用`Form.create`处理后的表单具有自动收集数据并校验的功能,但如果您不需要这个功能，或者默认的行为无法满足业务需求，可以选择不使用 `Form.create` 并自行处理数据。

那我们肯定需要,因为其灵魂就在于其,那么我们抛开布局,单纯来看一下,`Form.create`是如何实现

#### 哪里是核心？

​	`form.create`调用`rc-form`的`createDOMForm`,而`createDOMForm`则直接调用了`createBaseForm`,所以不严谨来说的话，`ant-design`的`form.create`我们可以认为只是`createBaseForm`别名，所以我们来看看`createBaseForm`是如何实现的？

#### React-form实现的双向绑定与Vue

> ​	经过 `Form.create` 包装的组件将会自带 `this.props.form` 属性，

​	而`form`属性中提供的`getFieldDecorator`则是实现数据自动收集的核心;

​	说到这个话题,我们跑一下题,目前国内前端呈现`React`与`Vue`双足鼎立的态势,ng的使用量是远远低于这两大框架的,可能有人会说了,不是说`Form`组件吗?又要开始写`娱乐圈`了?并不是,

​	我们来思考一下,简单了解过`Vue`的都能知道`Vue`的核心卖点是数据的双向绑定,而`React`则多为单向数据流,写两个简单的例子

```js
let data = 0

//Vue
<input v-model = "data"/>
    
//React
<input value={data} onChange={(e) => data = e.target.value}/>
```

​	假设`data`在这两个例子中都是被观测的,那么`Vue`和`React`的例子都实现了对`data`的双向绑定,也就是说,`Vue`总的来说也是单向数据流,在这两个例子中不同的是绑定方式,不同的是`Vue`提供了`v-model`语法糖,而`React`则给我们提供了更细粒度处理事件的机会,但是`Vue`也可以写成这种方式,只不过是`v-model`代劳了而已,那么到这里,我们回头再看看`getFieldDecorator`是不是跟`v-model`越看越像呢,

```js
getFieldDecorator('name',{})(
    <input/>
)
<input v-model = "data"/>
```

那么是不是`Form`是对`React`在表单上类似于`Vue`的另类实现呢?

是,也不是

#### 是与不是

​	为什么说是呢,因为`getFieldDecorator`默认绑定了`value`和`onChange`事件,这和`Vue`是相同的思想,那么不是又在哪里呢,我们回想一下,通过`Vue`的创建表单跟我们使用`antd`最大区别在哪里? 在于表单项,诚然`Vue`从底层API为我们实现了数据与UI的双向绑定,但是我们还是要自己去管理数据的,当然现在的现代框架提出的思想就是用数据去驱动UI,但是当我们使用`Form`时我们没有必要去管理它的数据,我们只需要在我们要提交数据时获取到`Form`中的值就可以了,这种更符合组件的思想,局部的数据,存储在局部的组件中,如果用这种思想来看Vue的`Form`的话,就会觉得`Form`的数据被提升到了父组件来进行管理.这对我们来说是不必要的

>当然,这是我的个人看法,如果Vue的相关组件库有类似实现,请各位不吝指正,同时此观点只是个人看法,希望各位不要认为我的思想就是对的

#### Form.create的具体实现

​	那我们说了这么多看一下其具体实现,因为我们在使用`Form`组件是使用最多的就是`getFieldDecorator`方法,所以我们首先来看一下他的实现原理

```js
getFieldDecorator(name, fieldOption) {
        const props = this.getFieldProps(name, fieldOption);
        return (fieldElem) => {
          // We should put field in record if it is rendered
          this.renderFields[name] = true;

          const fieldMeta = this.fieldsStore.getFieldMeta(name);
          const originalProps = fieldElem.props;
          fieldMeta.originalProps = originalProps;
          fieldMeta.ref = fieldElem.ref;
          return React.cloneElement(fieldElem, {
            ...props,
            ...this.fieldsStore.getFieldValuePropValue(fieldMeta),
          });
        };
  },
```

看一下这个函数的构成,调用`getFieldProps`方法构建`props`,随后将`props`跟其他相关配置挂载到传入的`ReactNode`上,所以从此来看,主要的逻辑配置在`getFieldProps`方法上,我 们来看一下`getFieldProps`的实现

```js
 getFieldProps(name, usersFieldOption = {}) {

        const fieldOption = {
          name,
          trigger: DEFAULT_TRIGGER,
          valuePropName: 'value',
          validate: [],
          ...usersFieldOption,
        };

        const {
          rules,
          trigger,
          validateTrigger = trigger,
          validate,
        } = fieldOption;

        const fieldMeta = this.fieldsStore.getFieldMeta(name);
        if ('initialValue' in fieldOption) {
          fieldMeta.initialValue = fieldOption.initialValue;
        }

        const inputProps = {
          ...this.fieldsStore.getFieldValuePropValue(fieldOption),
          ref: this.getCacheBind(name, `${name}__ref`, this.saveRef),
        };
        if (fieldNameProp) {
          inputProps[fieldNameProp] = formName ? `${formName}_${name}` : name;
        }

        const validateRules = normalizeValidateRules(validate, rules, validateTrigger);
        const validateTriggers = getValidateTriggers(validateRules);
        validateTriggers.forEach((action) => {
          if (inputProps[action]) return;
          inputProps[action] = this.getCacheBind(name, action, this.onCollectValidate);
        });

        // make sure that the value will be collect
        if (trigger && validateTriggers.indexOf(trigger) === -1) {
          inputProps[trigger] = this.getCacheBind(name, trigger, this.onCollect);
        }

        const meta = {
          ...fieldMeta,
          ...fieldOption,
          validate: validateRules,
        };
        this.fieldsStore.setFieldMeta(name, meta);
        if (fieldMetaProp) {
          inputProps[fieldMetaProp] = meta;
        }

        if (fieldDataProp) {
          inputProps[fieldDataProp] = this.fieldsStore.getField(name);
        }

        // This field is rendered, record it
        this.renderFields[name] = true;

        return inputProps;
      },
```

我删除了部分主要逻辑无关代码,我们看一下整个函数的思路,该函数首先进行了默认值的配置,如果未配置`trigger`和`valuePropName`则使用默认值,随后调用`fieldsStore.getFieldMeta`,`fieldsStore`在整个`form`中尤为关键,其作用是作为一个数据中心,让我们免除了手动去维护`form`中绑定的各个值,同时也是我刚才说的局部的数据存储于局部的组件思想.那么我们看一下`fieldsStore.getFieldMeta`做了什么

```js
//getFieldMeta在src/createFieldsStore下
  getFieldMeta(name) {
    this.fieldsMeta[name] = this.fieldsMeta[name] || {};
    return this.fieldsMeta[name];
  }
```



它的作用和它的名字一样是根据`name`获取`FieldMeta`,如果没有则创建,所以我们想象一下,整个`form`则会根据每个`field`的`name`值去创建索引表,现在我们知道在初始化情况下它返回的为空对象,

继续往下则是获取`initialValue`,关于这个可以看一下`antd form`的文档继续往后,下面到了最重要的`inputProps`构建环节,首先调用`getFieldValuePropValue`去获取`field`初始值,随后创建`ref`函数,我们暂时略过,我们来看一下最重要的数据收集

```js
const validateRules = normalizeValidateRules(validate, rules, validateTrigger);
const validateTriggers = getValidateTriggers(validateRules);
validateTriggers.forEach((action) => {
    if (inputProps[action]) return;
    inputProps[action] = this.getCacheBind(name, action, this.onCollectValidate);
});

// make sure that the value will be collect
if (trigger && validateTriggers.indexOf(trigger) === -1) {
    inputProps[trigger] = this.getCacheBind(name, trigger, this.onCollect);
}
```

我们着重来看一下这一部分代码,根据名称我们来看,`validateRules`应该是所有的校验规则,`validateTriggers`则是所有的校验规则触发事件的集合,我们来看一下这两个函数

```js
export function normalizeValidateRules(validate, rules, validateTrigger) {
  const validateRules = validate.map((item) => {
    const newItem = {
      ...item,
      trigger: item.trigger || [],
    };
    if (typeof newItem.trigger === 'string') {
      newItem.trigger = [newItem.trigger];
    }
    return newItem;
  });
  if (rules) {
    validateRules.push({
      trigger: validateTrigger ? [].concat(validateTrigger) : [],
      rules,
    });
  }
  return validateRules;
}

export function getValidateTriggers(validateRules) {
  return validateRules
    .filter(item => !!item.rules && item.rules.length)
    .map(item => item.trigger)
    .reduce((pre, curr) => pre.concat(curr), []);
}
```

我们看一下`normalizeValidateRules`函数,其会将`validate``rules`组合,返回一个数组,其内部的元素为一个个规则对象,并且每个元素都存在一个可以为空的`trigger`数组,并且将`validateTrigger`作为`rule`的`triggers`推入`validateRules`中,我们回回头看一下`validateTrigger`,

```js
 const fieldOption = {
     name,
     trigger: DEFAULT_TRIGGER,
     valuePropName: 'value',
     validate: [],
     ...usersFieldOption,
 };

const {
    rules,
    trigger,
    validateTrigger = trigger,
    validate,
} = fieldOption;
```

看一下这两个赋值,取值的语句,当我们没有配置`trigger`时使用`DEFAULT_TRIGGER`作为收集值的触发事件也就是`onChange`而当我们没有设置`validateTrigger`的时候使用`trigger`,这样说可能有点绕,简单点说,当我们配置了`validateTrigger`也就是验证触发函数时使用用户配置,未配置则使用用户配置的`trigger`,如果`trigger`用户都没有配置则全部使用默认配置也就是`onChange`,回过头来继续看着两个函数,`getValidateTriggers`则是将所有触发事件统一收集至一个数组,随后将所有`validateTriggers`中的事件都绑定上同一个处理函数,也就是接来下要说

```js
 validateTriggers.forEach((action) => {
          if (inputProps[action]) return;
          inputProps[action] = this.getCacheBind(name, action, this.onCollectValidate);
        });
```

我们看到,不管`validateTriggers`中哪一种事件被触发都会通过`this.getCacheBind(name, action, this.onCollectValidate);`来进行处理,首先来看一下`getCacheBind`

```js
getCacheBind(name, action, fn) {
        if (!this.cachedBind[name]) {
          this.cachedBind[name] = {};
        }
        const cache = this.cachedBind[name];
        if (!cache[action] || cache[action].oriFn !== fn) {
          cache[action] = {
            fn: fn.bind(this, name, action),
            oriFn: fn,
          };
        }
        return cache[action].fn;
      },
```

我们可以看到`getCacheBind`只是做了一下`bind`,真正的处理函数则是` this.onCollectValidate`,那我们来看一下` this.onCollectValidate`做了什么?

```js
onCollectValidate(name_, action, ...args) {
    const { field, fieldMeta } = this.onCollectCommon(name_, action, args);
    const newField = {
        ...field,
        dirty: true,
    };

    this.fieldsStore.setFieldsAsDirty();

    this.validateFieldsInternal([newField], {
        action,
        options: {
            firstFields: !!fieldMeta.validateFirst,
        },
    });
},
```

当`onCollectValidate`被调用,也就是数据校验函数被触发时主要做了四件事情,我们一条一条的来看

```js
onCollectCommon(name, action, args) {
    const fieldMeta = this.fieldsStore.getFieldMeta(name);
    if (fieldMeta[action]) {
        fieldMeta[action](...args);
    } else if (fieldMeta.originalProps && fieldMeta.originalProps[action]) {
        fieldMeta.originalProps[action](...args);
    }
    const value = fieldMeta.getValueFromEvent ?
          fieldMeta.getValueFromEvent(...args) :
    getValueFromEvent(...args);
    if (onValuesChange && value !== this.fieldsStore.getFieldValue(name)) {
        const valuesAll = this.fieldsStore.getAllValues();
        const valuesAllSet = {};
        valuesAll[name] = value;
        Object.keys(valuesAll).forEach(key => set(valuesAllSet, key, valuesAll[key]));
        onValuesChange({
            [formPropName]: this.getForm(),
            ...this.props
        }, set({}, name, value), valuesAllSet);
    }
    const field = this.fieldsStore.getField(name);
    return ({ name, field: { ...field, value, touched: true }, fieldMeta });
},
```

我们可以看出`onCollectCommon`主要是获取了包装元素新的值,随后将其包装在对象中返回,返回后将其组装为一个新的名为`newField`的对象,执行`fieldsStore.setFieldsAsDirty`,而`fieldsStore.setFieldsAsDirty`则是标记校验状态,我们暂且略过,随后执行`validateFieldsInternal`我们看一下`validateFieldsInternal`

```js
validateFieldsInternal(fields, {
        fieldNames,
        action,
        options = {},
      }, callback) {
        const allFields = {};
        fields.forEach((field) => {
          const name = field.name;
          if (options.force !== true && field.dirty === false) {
            if (field.errors) {
              set(alreadyErrors, name, { errors: field.errors });
            }
            return;
          }
          const fieldMeta = this.fieldsStore.getFieldMeta(name);
          const newField = {
            ...field,
          };
          newField.errors = undefined;
          newField.validating = true;
          newField.dirty = true;
          allRules[name] = this.getRules(fieldMeta, action);
          allValues[name] = newField.value;
          allFields[name] = newField;
        });
        this.setFields(allFields);
        // in case normalize
       ...dosometing
      },
```

因为`validateFieldsInternal`主要篇幅为调用`AsyncValidator`进行异步校验,我们暂时略过只看数据收集部分,

我们看到起最后调用了`this.setFields(allFields);`并传入了新的值,

看一下`setFields`

```js
setFields(maybeNestedFields, callback) {
        const fields = this.fieldsStore.flattenRegisteredFields(maybeNestedFields);
        this.fieldsStore.setFields(fields);
        if (onFieldsChange) {
          const changedFields = Object.keys(fields)
            .reduce((acc, name) => set(acc, name, this.fieldsStore.getField(name)), {});
          onFieldsChange({
            [formPropName]: this.getForm(),
            ...this.props
          }, changedFields, this.fieldsStore.getNestedAllFields());
        }
        this.forceUpdate(callback);
      },
```

我们可以看到,`setFields`首先对传入的只进行与初始化相似的验证,随后,将值存入`fieldsStore`,调用传入的`onFieldsChange`,之后调用`React.forceUpdate`更新视图.至此,我们简单的描述了整个流程,我们简述起具体流程则类似于Vue的`V-model`

获取初始值=>存储值数据中心也就是`fieldsStore`=>绑定收集值时机函数=>触发函数=>更新最新值至数据中心=>随后调用`forceUpdate`强制刷新视图.