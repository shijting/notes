### 使用场景

在实际业务中，我们经常会将系统分成很多层，在不同层会定义不同的实体。例如在 DAO 层有 PO（或者 Entity），在跨端调用的时候有 DTO，在和前端对接的时候有 VO。

大多数情况下，我们需要在这些实体进行转换，例如 DTO 转 PO 等。但是这种代码很呆板，因为仅仅是一个个字段赋值过去，所以在业务代码里面就会充斥着这种转化代码。

因此我们可以考虑设计一个复制器，完成不同类型直接的转换

### 行业分析

> 如果你知道有框架提供了类似功能，可以在这里描述，并且给出文档或者例子

#### Java BeanUtils

最为有名的应该是 Java 里面提供的 bean utils 依赖，里面支持在 bean 之间进行数据复制。同时 Java 还允许深度定制化复制的行为，例如允许忽略特定的字段，或者指定某个字段映射到另外一个字段。

### 可行方案

> 如果你有设计思路或者解决方案，请在这里提供。你可以提供多个方案，并且给出自己的选择

#### 反射

也就是我们这里采用的方案，即一个个字段使用反射复制过去。这里要考虑复杂类型字段，以及组合。





### 做法一：使用前缀树+反射



#### 为什么使用前缀树？

`Field` 比`FieldByName` 的性能快三十多倍

我对 `Field` 和 `FieldByName` 两个做了性能测试，效果如下：

```go
func BenchmarkFieldIndexOrName(b *testing.B) {
	tm := TestModel{}
	val := reflect.ValueOf(tm)
	b.Run("by index", func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			// 随便取一个，差异不大
			_ = val.Field(3)
		}
	})

	b.Run("by name", func(b *testing.B) {
		for i := 0; i < b.N; i++ {
			// 随便取一个，差异不大
			_ = val.FieldByName("Age")
		}
	})
}
```

在我电脑上执行测试 `go test -bench=BenchmarkFieldIndexOrName -benchmem -benchtime=10000x`，得到的结果是：

```
BenchmarkFieldIndexOrName/by_index-12       10000      2.840 ns/op           0 B/op          0 allocs/op      
BenchmarkFieldIndexOrName/by_name-12        10000      71.07 ns/op            8 B/op          1 allocs/op      
PASS
```

可以看到，by index 远比 by name 快。

errors.go
```go
import (
	"fmt"
	"reflect"
)

// newErrTypeError copier 不支持的类型
func newErrTypeError(typ reflect.Type) error {
	return fmt.Errorf("ekit: copier 入口只支持 Struct 不支持类型 %v, 种类 %v", typ, typ.Kind())
}

// newErrKindNotMatchError 字段类型不匹配
func newErrKindNotMatchError(src, dst reflect.Kind, field string) error {
	return fmt.Errorf("ekit: 字段 %s 的 Kind 不匹配, src: %v, dst: %v", field, src, dst)
}

// newErrTypeNotMatchError 字段不匹配
func newErrTypeNotMatchError(src, dst reflect.Type, field string) error {
	return fmt.Errorf("ekit: 字段 %s 的 Type 不匹配, src: %v, dst: %v", field, src, dst)
}

// newErrMultiPointer
func newErrMultiPointer(field string) error {
	return fmt.Errorf("ekit: 字段 %s 是多级指针", field)
}

```

copy.go
```go
// Copier 复制数据
// 1. 深拷贝亦或是浅拷贝，取决于具体的实现。每个实现都要声明清楚这一点；
// 2. Src 和 Dst 都必须是普通的结构体，支持组合
// 3. 只复制公共字段
// 这种设计设计，即使用 *Src 和 *Dst 可能加剧内存逃逸
type Copier[Src any, Dst any] interface {
	// CopyTo 将 src 中的数据复制到 dst 中
	CopyTo(src *Src, dst *Dst) error
	// Copy 将创建一个 Dst 的实例，并且将 Src 中的数据复制过去
	Copy(src *Src) (*Dst, error)
}
```

reflect_copier.go
```go
// ReflectCopier 基于反射的实现
// ReflectCopier 是浅拷贝
type ReflectCopier[Src any, Dst any] struct {

	// rootField 字典树的根节点
	rootField fieldNode
}

// fieldNode 字段的前缀树
type fieldNode struct {
	// 当前节点的名字
	name string

	// 当前 Struct 的子节点, 如果为叶子节点, 则没有子节点
	fields []fieldNode

    // 在 source 的 index。在反射中使用Field() 比 FieldByName 快三十倍
	srcIndex int

	// 在 dst 的 index
	dstIndex int

	// 是否为叶子节点, 如果为叶子节点, 应该直接进行拷贝该字段
	isLeaf bool
}

// NewReflectCopier 如果类型不匹配, 创建时直接检查报错.
func NewReflectCopier[Src any, Dst any]() (*ReflectCopier[Src, Dst], error) {
	src := new(Src)
	srcTyp := reflect.TypeOf(src).Elem()
	dst := new(Dst)
	dstTyp := reflect.TypeOf(dst).Elem()
	root := fieldNode{
		isLeaf: false,
		fields: []fieldNode{},
	}
	if srcTyp.Kind() != reflect.Struct {
		return nil, newErrTypeError(srcTyp)
	}
	if dstTyp.Kind() != reflect.Struct {
		return nil, newErrTypeError(dstTyp)
	}
	if err := createFieldNodes(&root, srcTyp, dstTyp); err != nil {
		return nil, err
	}

	copier := &ReflectCopier[Src, Dst]{
		rootField: root,
	}
	return copier, nil
}

// createFieldNodes 递归创建 field 的前缀树, srcTyp 和 dstTyp 只能是结构体
func createFieldNodes(root *fieldNode, srcTyp, dstTyp reflect.Type) error {

	fieldMap := map[string]int{}
	for i := 0; i < srcTyp.NumField(); i++ {
		srcFieldTypStruct := srcTyp.Field(i)
		if !srcFieldTypStruct.IsExported() {
			continue
		}
		fieldMap[srcFieldTypStruct.Name] = i
	}

	for dstIndex := 0; dstIndex < dstTyp.NumField(); dstIndex++ {

		dstFieldTypStruct := dstTyp.Field(dstIndex)
		if !dstFieldTypStruct.IsExported() {
			continue
		}
		srcIndex, ok := fieldMap[dstFieldTypStruct.Name]
		if !ok {
			continue
		}
		srcFieldTypStruct := srcTyp.Field(srcIndex)
		if srcFieldTypStruct.Type.Kind() != dstFieldTypStruct.Type.Kind() {
			return newErrKindNotMatchError(srcFieldTypStruct.Type.Kind(), dstFieldTypStruct.Type.Kind(), dstFieldTypStruct.Name)
		}

		if srcFieldTypStruct.Type.Kind() == reflect.Pointer {
			if srcFieldTypStruct.Type.Elem().Kind() != dstFieldTypStruct.Type.Elem().Kind() {
				return newErrKindNotMatchError(srcFieldTypStruct.Type.Kind(), dstFieldTypStruct.Type.Kind(), dstFieldTypStruct.Name)
			}
			if srcFieldTypStruct.Type.Elem().Kind() == reflect.Pointer {
				return newErrMultiPointer(dstFieldTypStruct.Name)
			}
		}

		child := fieldNode{
			fields:   []fieldNode{},
			srcIndex: srcIndex,
			dstIndex: dstIndex,
			isLeaf:   false,
			name:     dstFieldTypStruct.Name,
		}

		fieldSrcTyp := srcFieldTypStruct.Type
		fieldDstTyp := dstFieldTypStruct.Type
		if fieldSrcTyp.Kind() == reflect.Pointer {
			fieldSrcTyp = fieldSrcTyp.Elem()
			fieldDstTyp = fieldDstTyp.Elem()
		}

		if isShadowCopyType(fieldSrcTyp.Kind()) {
			// 内置类型，但不匹配，如别名、map和slice
			if fieldSrcTyp != fieldDstTyp {
				return newErrTypeNotMatchError(srcFieldTypStruct.Type, dstFieldTypStruct.Type, dstFieldTypStruct.Name)
			}
			// 说明当前节点是叶子节点, 直接拷贝
			child.isLeaf = true
		} else if fieldSrcTyp.Kind() == reflect.Struct {
			if err := createFieldNodes(&child, fieldSrcTyp, fieldDstTyp); err != nil {
				return err
			}
		} else {
			// 不是我们能复制的类型, 直接跳过
			continue
		}

		root.fields = append(root.fields, child)
	}
	return nil
}

func (r *ReflectCopier[Src, Dst]) Copy(src *Src) (*Dst, error) {
	dst := new(Dst)
	err := r.CopyTo(src, dst)
	return dst, err
}

// CopyTo 执行复制
// 执行复制的逻辑是：
// 1. 按照字段的映射关系进行匹配
// 2. 如果 Src 和 Dst 中匹配的字段，其类型是基本类型（及其指针）或者内置类型（及其指针），并且类型一样，则直接用 Src 的值
// 3. 如果 Src 和 Dst 中匹配的字段，其类型都是结构体，或者都是结构体指针，则会深入复制
// 4. 否则，忽略字段
func (r *ReflectCopier[Src, Dst]) CopyTo(src *Src, dst *Dst) error {
	return r.copyToWithTree(src, dst)
}

func (r *ReflectCopier[Src, Dst]) copyToWithTree(src *Src, dst *Dst) error {
	srcTyp := reflect.TypeOf(src)
	dstTyp := reflect.TypeOf(dst)
	srcValue := reflect.ValueOf(src)
	dstValue := reflect.ValueOf(dst)

	return r.copyTreeNode(srcTyp, srcValue, dstTyp, dstValue, &r.rootField)
}

func (r *ReflectCopier[Src, Dst]) copyTreeNode(srcTyp reflect.Type, srcValue reflect.Value, dstType reflect.Type, dstValue reflect.Value, root *fieldNode) error {
	if srcValue.Kind() == reflect.Pointer {
		if srcValue.IsNil() {
			return nil
		}
		if dstValue.IsNil() {
			dstValue.Set(reflect.New(dstType.Elem()))
		}
		srcValue = srcValue.Elem()
		srcTyp = srcTyp.Elem()

		dstValue = dstValue.Elem()
		dstType = dstType.Elem()
	}
	// 执行拷贝
	if root.isLeaf {
		if dstValue.CanSet() {
			dstValue.Set(srcValue)
		}
		return nil
	}

	for i := range root.fields {
		child := &root.fields[i]
		childSrcTyp := srcTyp.Field(child.srcIndex)
		childSrcValue := srcValue.Field(child.srcIndex)

		childDstTyp := dstType.Field(child.dstIndex)
		childDstValue := dstValue.Field(child.dstIndex)
		if err := r.copyTreeNode(childSrcTyp.Type, childSrcValue, childDstTyp.Type, childDstValue, child); err != nil {
			return err
		}
	}
	return nil
}

func isShadowCopyType(kind reflect.Kind) bool {
	switch kind {
	case reflect.Bool,
		reflect.Int,
		reflect.Int8,
		reflect.Int16,
		reflect.Int32,
		reflect.Int64,
		reflect.Uint,
		reflect.Uint8,
		reflect.Uint16,
		reflect.Uint32,
		reflect.Uint64,
		reflect.Uintptr,
		reflect.Float32,
		reflect.Float64,
		reflect.Complex64,
		reflect.Complex128,
		reflect.String,
		reflect.Slice,
		reflect.Map,
		reflect.Chan,
		reflect.Array:
		return true
	}
	return false
}

```


reflect_copier_test.go
```go
import (
	"reflect"
	"testing"

	"github.com/stretchr/testify/assert"
)

func TestReflectCopier_Copy(t *testing.T) {
	testCases := []struct {
		name     string
		copyFunc func() (any, error)
		wantDst  any
		wantErr  error
	}{
		{
			name: "simple struct",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SimpleSrc, SimpleDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SimpleSrc{
					Name:    "大明",
					Age:     ToPtr[int](18),
					Friends: []string{"Tom", "Jerry"},
				})
			},
			wantDst: &SimpleDst{
				Name:    "大明",
				Age:     ToPtr[int](18),
				Friends: []string{"Tom", "Jerry"},
			},
		},
		{
			name: "基础类型的 struct",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[BasicSrc, BasicDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&BasicSrc{
					Name:    "大明",
					Age:     10,
					CNumber: complex(1, 2),
				})
			},
			wantDst: &BasicDst{
				Name:    "大明",
				Age:     10,
				CNumber: complex(1, 2),
			},
		},
		{
			name: "src 是基础类型",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[int, int]()
				if err != nil {
					return nil, err
				}
				i := 10
				return copier.Copy(&i)
			},
			wantErr: newErrTypeError(reflect.TypeOf(10)),
		},
		{
			name: "dst 是基础类型",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SimpleSrc, string]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SimpleSrc{
					Name:    "大明",
					Age:     ToPtr[int](18),
					Friends: []string{"Tom", "Jerry"},
				})
			},
			wantErr: newErrTypeError(reflect.TypeOf("")),
		},
		{
			name: "接口类型",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[InterfaceSrc, InterfaceDst]()
				if err != nil {
					return nil, err
				}
				i := InterfaceSrc(10)
				return copier.Copy(&i)
			},
			wantErr: newErrTypeError(reflect.TypeOf(new(InterfaceSrc)).Elem()),
		},
		{
			name: "simple struct 空切片, 空指针",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SimpleSrc, SimpleDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SimpleSrc{
					Name: "大明",
				})
			},
			wantDst: &SimpleDst{
				Name: "大明",
			},
		},
		{
			name: "组合 struct ",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[EmbedSrc, EmbedDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&EmbedSrc{
					SimpleSrc: SimpleSrc{
						Name:    "xiaoli",
						Age:     ToPtr[int](19),
						Friends: []string{},
					},
					BasicSrc: &BasicSrc{
						Name:    "xiaowang",
						Age:     20,
						CNumber: complex(2, 2),
					},
				})
			},
			wantDst: &EmbedDst{
				SimpleSrc: SimpleSrc{
					Name:    "xiaoli",
					Age:     ToPtr[int](19),
					Friends: []string{},
				},
				BasicSrc: &BasicSrc{
					Name:    "xiaowang",
					Age:     20,
					CNumber: complex(2, 2),
				},
			},
		},
		{
			name: "复杂 Struct",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[ComplexSrc, ComplexDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&ComplexSrc{
					Simple: SimpleSrc{
						Name:    "xiaohong",
						Age:     ToPtr[int](18),
						Friends: []string{"ha", "ha", "le"},
					},
					Embed: &EmbedSrc{
						SimpleSrc: SimpleSrc{
							Name:    "xiaopeng",
							Age:     ToPtr[int](88),
							Friends: []string{"la", "ha", "le"},
						},
						BasicSrc: &BasicSrc{
							Name:    "wang",
							Age:     22,
							CNumber: complex(2, 1),
						},
					},
					BasicSrc: BasicSrc{
						Name:    "wang11",
						Age:     22,
						CNumber: complex(2, 1),
					},
				})
			},
			wantDst: &ComplexDst{
				Simple: SimpleDst{
					Name:    "xiaohong",
					Age:     ToPtr[int](18),
					Friends: []string{"ha", "ha", "le"},
				},
				Embed: &EmbedDst{
					SimpleSrc: SimpleSrc{
						Name:    "xiaopeng",
						Age:     ToPtr[int](88),
						Friends: []string{"la", "ha", "le"},
					},
					BasicSrc: &BasicSrc{
						Name:    "wang",
						Age:     22,
						CNumber: complex(2, 1),
					},
				},
				BasicSrc: BasicSrc{
					Name:    "wang11",
					Age:     22,
					CNumber: complex(2, 1),
				},
			},
		},
		{
			name: "特殊类型",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SpecialSrc, SpecialDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SpecialSrc{
					Arr: [3]float32{1, 2, 3},
					M: map[string]int{
						"ha": 1,
						"o":  2,
					},
				})
			},
			wantDst: &SpecialDst{
				Arr: [3]float32{1, 2, 3},
				M: map[string]int{
					"ha": 1,
					"o":  2,
				},
			},
		},
		{
			name: "复杂 Struct 不匹配",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[NotMatchSrc, NotMatchDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&NotMatchSrc{
					Simple: SimpleSrc{
						Name:    "xiaohong",
						Age:     ToPtr[int](18),
						Friends: []string{"ha", "ha", "le"},
					},
					Embed: &EmbedSrc{
						SimpleSrc: SimpleSrc{
							Name:    "xiaopeng",
							Age:     ToPtr[int](88),
							Friends: []string{"la", "ha", "le"},
						},
						BasicSrc: &BasicSrc{
							Name:    "wang",
							Age:     22,
							CNumber: complex(2, 1),
						},
					},
					BasicSrc: BasicSrc{
						Name:    "wang11",
						Age:     22,
						CNumber: complex(2, 1),
					},
					S: struct{ A string }{A: "a"},
				})
			},
			wantErr: newErrKindNotMatchError(reflect.String, reflect.Int, "A"),
		},
		{
			name: "多重指针",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[MultiPtrSrc, MultiPtrDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&MultiPtrSrc{
					Name:    "a",
					Age:     ToPtr[*int](ToPtr[int](10)),
					Friends: nil,
				})
			},
			wantErr: newErrMultiPointer("Age"),
		},
		{
			name: "src 有额外字段",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[DiffSrc, DiffDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&DiffSrc{
					A: "xiaowang",
					B: 100,
					c: SimpleSrc{
						Name: "66",
						Age:  ToPtr[int](100),
					},
					F: BasicSrc{
						Name:    "good name",
						Age:     200,
						CNumber: complex(2, 2),
					},
				})
			},
			wantDst: &DiffDst{
				A: "xiaowang",
				B: 100,
				d: SimpleSrc{},
				G: BasicSrc{},
			},
		},
		{
			name: "dst 有额外字段",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[DiffSrc, DiffDst]()
				if err != nil {
					return nil, err
				}
				dst := &DiffDst{
					A: "66",
					B: 1,
					d: SimpleSrc{
						Name: "wodemingzi",
						Age:  ToPtr(int(10)),
					},
					G: BasicSrc{
						Name:    "nidemingzi",
						Age:     23,
						CNumber: complex(1, 2),
					},
				}
				err = copier.CopyTo(&DiffSrc{
					A: "xiaowang",
					B: 100,
					c: SimpleSrc{
						Name: "66",
						Age:  ToPtr[int](100),
					},
					F: BasicSrc{
						Name:    "good name",
						Age:     200,
						CNumber: complex(2, 2),
					},
				}, dst)
				return dst, err
			},
			wantDst: &DiffDst{
				A: "xiaowang",
				B: 100,
				d: SimpleSrc{
					Name: "wodemingzi",
					Age:  ToPtr(int(10)),
				},
				G: BasicSrc{
					Name:    "nidemingzi",
					Age:     23,
					CNumber: complex(1, 2),
				},
			},
		},
		{
			name: "跨层级别匹配",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SimpleSrc, SimpleEmbedDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SimpleSrc{})
			},
			wantDst: &SimpleEmbedDst{},
		},
		{
			name: "成员为结构体数组",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[ArraySrc, ArrayDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&ArraySrc{
					A: []SimpleSrc{
						{
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
						{
							Name:    "小明",
							Age:     ToPtr[int](8),
							Friends: []string{"Tom"},
						},
					},
				})
			},
			wantDst: &ArrayDst{
				A: []SimpleSrc{
					{
						Name:    "大明",
						Age:     ToPtr[int](18),
						Friends: []string{"Tom", "Jerry"},
					},
					{
						Name:    "小明",
						Age:     ToPtr[int](8),
						Friends: []string{"Tom"},
					},
				},
			},
		},
		{
			name: "成员为结构体数组，结构体不同",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[ArraySrc, ArrayDst1]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&ArraySrc{
					A: []SimpleSrc{
						{
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
						{
							Name:    "小明",
							Age:     ToPtr[int](8),
							Friends: []string{"Tom"},
						},
					},
				})
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new([]SimpleSrc)).Elem(), reflect.TypeOf(new([]SimpleDst)).Elem(), "A"),
		},
		{
			name: "成员为map结构体",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[MapSrc, MapDst]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&MapSrc{
					A: map[string]SimpleSrc{
						"a": {
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
					},
				})
			},
			wantDst: &MapDst{
				A: map[string]SimpleSrc{
					"a": {
						Name:    "大明",
						Age:     ToPtr[int](18),
						Friends: []string{"Tom", "Jerry"},
					},
				},
			},
		},
		{
			name: "成员为不同的map结构体",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[MapSrc, MapDst1]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&MapSrc{
					A: map[string]SimpleSrc{
						"a": {
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
					},
				})
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new(map[string]SimpleSrc)).Elem(), reflect.TypeOf(new(map[string]SimpleDst)).Elem(), "A"),
		},
		{
			name: "成员有别名类型",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SpecialSrc1, SpecialDst1]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SpecialSrc1{
					A: 1,
				})
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new(int)).Elem(), reflect.TypeOf(new(aliasInt)).Elem(), "A"),
		},
		{
			name: "成员有别名类型1",
			copyFunc: func() (any, error) {
				copier, err := NewReflectCopier[SpecialSrc1, SpecialDst2]()
				if err != nil {
					return nil, err
				}
				return copier.Copy(&SpecialSrc1{
					A: 1,
				})
			},
			wantDst: &SpecialDst2{A: 1},
		},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T) {
			res, err := tc.copyFunc()
			assert.Equal(t, tc.wantErr, err)
			if err != nil {
				return
			}
			assert.Equal(t, tc.wantDst, res)
		})
	}
}

func ToPtr[T any](t T) *T {
	return &t
}

type BasicSrc struct {
	Name    string
	Age     int
	CNumber complex64
}

type BasicDst struct {
	Name    string
	Age     int
	CNumber complex64
}

type SimpleSrc struct {
	Name    string
	Age     *int
	Friends []string
}

type SimpleDst struct {
	Name    string
	Age     *int
	Friends []string
}

type EmbedSrc struct {
	SimpleSrc
	*BasicSrc
}

type EmbedDst struct {
	SimpleSrc
	*BasicSrc
}

type ComplexSrc struct {
	Simple SimpleSrc
	Embed  *EmbedSrc
	BasicSrc
}

type ComplexDst struct {
	Simple SimpleDst
	Embed  *EmbedDst
	BasicSrc
}

type SpecialSrc struct {
	Arr [3]float32
	M   map[string]int
}

type SpecialDst struct {
	Arr [3]float32
	M   map[string]int
}

type InterfaceSrc interface {
}

type InterfaceDst interface {
}

type NotMatchSrc struct {
	Simple SimpleSrc
	Embed  *EmbedSrc
	BasicSrc
	S struct {
		A string
	}
}

type NotMatchDst struct {
	Simple SimpleDst
	Embed  *EmbedDst
	BasicSrc
	S struct {
		A int
	}
}

type MultiPtrSrc struct {
	Name    string
	Age     **int
	Friends []string
}

type MultiPtrDst struct {
	Name    string
	Age     **int
	Friends []string
}

type DiffSrc struct {
	A string
	B int
	c SimpleSrc
	F BasicSrc
}
type DiffDst struct {
	A string
	B int
	d SimpleSrc
	G BasicSrc
}

type SimpleEmbedDst struct {
	SimpleSrc
}

type ArraySrc struct {
	A []SimpleSrc
}

type ArrayDst struct {
	A []SimpleSrc
}

type ArrayDst1 struct {
	A []SimpleDst
}

type MapSrc struct {
	A map[string]SimpleSrc
}

type MapDst struct {
	A map[string]SimpleSrc
}

type MapDst1 struct {
	A map[string]SimpleDst
}

type SpecialSrc1 struct {
	A int
}

type aliasInt int
type SpecialDst1 struct {
	A aliasInt
}

type aliasInt1 = int
type SpecialDst2 struct {
	A aliasInt1
}

func BenchmarkReflectCopier_Copy(b *testing.B) {
	copier, err := NewReflectCopier[SimpleSrc, SimpleDst]()
	if err != nil {
		b.Fatal(err)
	}
	for i := 1; i <= b.N; i++ {
		_, _ = copier.Copy(&SimpleSrc{
			Name:    "大明",
			Age:     ToPtr[int](18),
			Friends: []string{"Tom", "Jerry"},
		})
	}
}

func BenchmarkReflectCopier_CopyComplexStruct(b *testing.B) {
	copier, err := NewReflectCopier[ComplexSrc, ComplexDst]()
	if err != nil {
		b.Fatal(err)
	}
	for i := 1; i <= b.N; i++ {
		_, _ = copier.Copy(&ComplexSrc{
			Simple: SimpleSrc{
				Name:    "xiaohong",
				Age:     ToPtr[int](18),
				Friends: []string{"ha", "ha", "le"},
			},
			Embed: &EmbedSrc{
				SimpleSrc: SimpleSrc{
					Name:    "xiaopeng",
					Age:     ToPtr[int](88),
					Friends: []string{"la", "ha", "le"},
				},
				BasicSrc: &BasicSrc{
					Name:    "wang",
					Age:     22,
					CNumber: complex(2, 1),
				},
			},
			BasicSrc: BasicSrc{
				Name:    "wang11",
				Age:     22,
				CNumber: complex(2, 1),
			},
		})
	}
}

```



### 做法二，只使用反射

pure_reflect_copier.go
```go
// CopyTo 复制结构体, 纯递归实现. src 和 dst 都必须是结构体的指针
func CopyTo(src any, dst any) error {
	srcPtrTyp := reflect.TypeOf(src)
	if srcPtrTyp.Kind() != reflect.Pointer {
		return newErrTypeError(srcPtrTyp)
	}
	srcTyp := srcPtrTyp.Elem()
	if srcTyp.Kind() != reflect.Struct {
		return newErrTypeError(srcTyp)
	}
	dstPtrTyp := reflect.TypeOf(dst)
	if dstPtrTyp.Kind() != reflect.Pointer {
		return newErrTypeError(dstPtrTyp)
	}
	dstTyp := dstPtrTyp.Elem()
	if dstTyp.Kind() != reflect.Struct {
		return newErrTypeError(dstTyp)
	}

	srcValue := reflect.ValueOf(src).Elem()
	dstValue := reflect.ValueOf(dst).Elem()

	return copyStruct(srcTyp, srcValue, dstTyp, dstValue)
}

func copyStruct(srcTyp reflect.Type, srcValue reflect.Value, dstTyp reflect.Type, dstValue reflect.Value) error {
	srcFieldNameIndex := make(map[string]int, 0)
	for i := 0; i < srcTyp.NumField(); i++ {
		fTyp := srcTyp.Field(i)
		if !fTyp.IsExported() {
			continue
		}
		srcFieldNameIndex[fTyp.Name] = i
	}

	for i := 0; i < dstTyp.NumField(); i++ {
		fTyp := dstTyp.Field(i)
		if !fTyp.IsExported() {
			continue
		}
		if idx, ok := srcFieldNameIndex[fTyp.Name]; ok {
			if err := copyStructField(srcTyp, srcValue, dstTyp, dstValue, idx, i); err != nil {
				return err
			}
		}
	}
	return nil
}

func copyStructField(
	srcTyp reflect.Type,
	srcValue reflect.Value,
	dstTyp reflect.Type,
	dstValue reflect.Value,
	srcFieldIndex int,
	dstFieldIndex int) error {

	srcFieldType := srcTyp.Field(srcFieldIndex)
	dstFieldType := dstTyp.Field(dstFieldIndex)
	if srcFieldType.Type.Kind() != dstFieldType.Type.Kind() {
		return newErrKindNotMatchError(srcFieldType.Type.Kind(), dstFieldType.Type.Kind(), srcFieldType.Name)
	}
	srcFieldValue := srcValue.Field(srcFieldIndex)
	dstFieldValue := dstValue.Field(dstFieldIndex)

	if srcFieldType.Type.Kind() == reflect.Pointer {
		if srcFieldValue.IsNil() {
			return nil
		}
		if dstFieldValue.IsNil() {
			dstFieldValue.Set(reflect.New(dstFieldType.Type.Elem()))
		}
		return copyData(srcFieldType.Type.Elem(), srcFieldValue.Elem(), dstFieldType.Type.Elem(), dstFieldValue.Elem(), srcFieldType.Name)
	}

	return copyData(srcFieldType.Type, srcFieldValue, dstFieldType.Type, dstFieldValue, srcFieldType.Name)
}

func copyData(
	srcTyp reflect.Type,
	srcValue reflect.Value,
	dstTyp reflect.Type,
	dstValue reflect.Value,
	fieldName string,
) error {
	if srcTyp.Kind() == reflect.Pointer {
		return newErrMultiPointer(fieldName)
	}
	if srcTyp.Kind() != dstTyp.Kind() {
		return newErrKindNotMatchError(srcTyp.Kind(), dstTyp.Kind(), fieldName)
	}

	if isShadowCopyType(srcTyp.Kind()) {
		// 内置类型，但不匹配，如别名、map和slice
		if srcTyp != dstTyp {
			return newErrTypeNotMatchError(srcTyp, dstTyp, fieldName)
		}
		if dstValue.CanSet() {
			dstValue.Set(srcValue)
		}
	} else if srcTyp.Kind() == reflect.Struct {
		return copyStruct(srcTyp, srcValue, dstTyp, dstValue)
	}
	return nil
}
```

pure_reflect_copier_test.go
```go
func TestReflectCopier_CopyTo(t *testing.T) {
	testCases := []struct {
		name     string
		copyFunc func() (any, error)
		wantDst  any
		wantErr  error
	}{
		{
			name: "simple struct",
			copyFunc: func() (any, error) {
				dst := &SimpleDst{}
				err := CopyTo(&SimpleSrc{
					Name:    "大明",
					Age:     ToPtr[int](18),
					Friends: []string{"Tom", "Jerry"},
				}, dst)
				return dst, err
			},
			wantDst: &SimpleDst{
				Name:    "大明",
				Age:     ToPtr[int](18),
				Friends: []string{"Tom", "Jerry"},
			},
		},
		{
			name: "基础类型的 struct",
			copyFunc: func() (any, error) {
				dst := &BasicDst{}
				err := CopyTo(&BasicSrc{
					Name:    "大明",
					Age:     10,
					CNumber: complex(1, 2),
				}, dst)
				return dst, err
			},
			wantDst: &BasicDst{
				Name:    "大明",
				Age:     10,
				CNumber: complex(1, 2),
			},
		},
		{
			name: "src 是基础类型",
			copyFunc: func() (any, error) {
				i := 10
				dst := ToPtr(int(0))
				err := CopyTo(&i, dst)
				return dst, err
			},
			wantErr: newErrTypeError(reflect.TypeOf(10)),
		},
		{
			name: "dst 是基础类型",
			copyFunc: func() (any, error) {

				dst := ToPtr("")
				err := CopyTo(&SimpleSrc{
					Name:    "大明",
					Age:     ToPtr[int](18),
					Friends: []string{"Tom", "Jerry"},
				}, dst)
				return dst, err
			},
			wantErr: newErrTypeError(reflect.TypeOf("")),
		},
		{
			name: "接口类型",
			copyFunc: func() (any, error) {
				i := InterfaceSrc(10)
				dst := ToPtr(InterfaceDst(10))
				err := CopyTo(&i, dst)
				return dst, err
			},
			wantErr: newErrTypeError(reflect.TypeOf(new(InterfaceSrc)).Elem()),
		},
		{
			name: "simple struct 空切片, 空指针",
			copyFunc: func() (any, error) {
				dst := &SimpleDst{}
				err := CopyTo(&SimpleSrc{
					Name: "大明",
				}, dst)
				return dst, err
			},
			wantDst: &SimpleDst{
				Name: "大明",
			},
		},
		{
			name: "组合 struct ",
			copyFunc: func() (any, error) {
				dst := &EmbedDst{}
				err := CopyTo(&EmbedSrc{
					SimpleSrc: SimpleSrc{
						Name:    "xiaoli",
						Age:     ToPtr[int](19),
						Friends: []string{},
					},
					BasicSrc: &BasicSrc{
						Name:    "xiaowang",
						Age:     20,
						CNumber: complex(2, 2),
					},
				}, dst)
				return dst, err
			},
			wantDst: &EmbedDst{
				SimpleSrc: SimpleSrc{
					Name:    "xiaoli",
					Age:     ToPtr[int](19),
					Friends: []string{},
				},
				BasicSrc: &BasicSrc{
					Name:    "xiaowang",
					Age:     20,
					CNumber: complex(2, 2),
				},
			},
		},
		{
			name: "复杂 Struct",
			copyFunc: func() (any, error) {
				dst := &ComplexDst{}
				err := CopyTo(&ComplexSrc{
					Simple: SimpleSrc{
						Name:    "xiaohong",
						Age:     ToPtr[int](18),
						Friends: []string{"ha", "ha", "le"},
					},
					Embed: &EmbedSrc{
						SimpleSrc: SimpleSrc{
							Name:    "xiaopeng",
							Age:     ToPtr[int](88),
							Friends: []string{"la", "ha", "le"},
						},
						BasicSrc: &BasicSrc{
							Name:    "wang",
							Age:     22,
							CNumber: complex(2, 1),
						},
					},
					BasicSrc: BasicSrc{
						Name:    "wang11",
						Age:     22,
						CNumber: complex(2, 1),
					},
				}, dst)
				return dst, err
			},
			wantDst: &ComplexDst{
				Simple: SimpleDst{
					Name:    "xiaohong",
					Age:     ToPtr[int](18),
					Friends: []string{"ha", "ha", "le"},
				},
				Embed: &EmbedDst{
					SimpleSrc: SimpleSrc{
						Name:    "xiaopeng",
						Age:     ToPtr[int](88),
						Friends: []string{"la", "ha", "le"},
					},
					BasicSrc: &BasicSrc{
						Name:    "wang",
						Age:     22,
						CNumber: complex(2, 1),
					},
				},
				BasicSrc: BasicSrc{
					Name:    "wang11",
					Age:     22,
					CNumber: complex(2, 1),
				},
			},
		},
		{
			name: "特殊类型",
			copyFunc: func() (any, error) {
				dst := &SpecialDst{}
				err := CopyTo(&SpecialSrc{
					Arr: [3]float32{1, 2, 3},
					M: map[string]int{
						"ha": 1,
						"o":  2,
					},
				}, dst)
				return dst, err
			},
			wantDst: &SpecialDst{
				Arr: [3]float32{1, 2, 3},
				M: map[string]int{
					"ha": 1,
					"o":  2,
				},
			},
		},
		{
			name: "复杂 Struct 不匹配",
			copyFunc: func() (any, error) {
				dst := &NotMatchDst{}
				err := CopyTo(&NotMatchSrc{
					Simple: SimpleSrc{
						Name:    "xiaohong",
						Age:     ToPtr[int](18),
						Friends: []string{"ha", "ha", "le"},
					},
					Embed: &EmbedSrc{
						SimpleSrc: SimpleSrc{
							Name:    "xiaopeng",
							Age:     ToPtr[int](88),
							Friends: []string{"la", "ha", "le"},
						},
						BasicSrc: &BasicSrc{
							Name:    "wang",
							Age:     22,
							CNumber: complex(2, 1),
						},
					},
					BasicSrc: BasicSrc{
						Name:    "wang11",
						Age:     22,
						CNumber: complex(2, 1),
					},
					S: struct{ A string }{A: "a"},
				}, dst)
				return dst, err
			},
			wantErr: newErrKindNotMatchError(reflect.String, reflect.Int, "A"),
		},
		{
			name: "多重指针",
			copyFunc: func() (any, error) {
				dst := &MultiPtrDst{}
				err := CopyTo(&MultiPtrSrc{
					Name:    "a",
					Age:     ToPtr[*int](ToPtr[int](10)),
					Friends: nil,
				}, dst)
				return dst, err
			},
			wantErr: newErrMultiPointer("Age"),
		},
		{
			name: "src 有额外字段",
			copyFunc: func() (any, error) {
				dst := &DiffDst{}
				err := CopyTo(&DiffSrc{
					A: "xiaowang",
					B: 100,
					c: SimpleSrc{
						Name: "66",
						Age:  ToPtr[int](100),
					},
					F: BasicSrc{
						Name:    "good name",
						Age:     200,
						CNumber: complex(2, 2),
					},
				}, dst)
				return dst, err
			},
			wantDst: &DiffDst{
				A: "xiaowang",
				B: 100,
				d: SimpleSrc{},
				G: BasicSrc{},
			},
		},
		{
			name: "dst 有额外字段",
			copyFunc: func() (any, error) {

				dst := &DiffDst{
					A: "66",
					B: 1,
					d: SimpleSrc{
						Name: "wodemingzi",
						Age:  ToPtr(int(10)),
					},
					G: BasicSrc{
						Name:    "nidemingzi",
						Age:     23,
						CNumber: complex(1, 2),
					},
				}
				err := CopyTo(&DiffSrc{
					A: "xiaowang",
					B: 100,
					c: SimpleSrc{
						Name: "66",
						Age:  ToPtr[int](100),
					},
					F: BasicSrc{
						Name:    "good name",
						Age:     200,
						CNumber: complex(2, 2),
					},
				}, dst)
				return dst, err
			},
			wantDst: &DiffDst{
				A: "xiaowang",
				B: 100,
				d: SimpleSrc{
					Name: "wodemingzi",
					Age:  ToPtr(int(10)),
				},
				G: BasicSrc{
					Name:    "nidemingzi",
					Age:     23,
					CNumber: complex(1, 2),
				},
			},
		},
		{
			name: "跨层级别匹配",
			copyFunc: func() (any, error) {
				dst := &SimpleEmbedDst{}
				err := CopyTo(&SimpleSrc{
					Name: "haha",
				}, dst)
				return dst, err
			},
			wantDst: &SimpleEmbedDst{},
		},
		{
			name: "成员为结构体数组",
			copyFunc: func() (any, error) {
				dst := &ArrayDst{}
				return dst, CopyTo(&ArraySrc{
					A: []SimpleSrc{
						{
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
						{
							Name:    "小明",
							Age:     ToPtr[int](8),
							Friends: []string{"Tom"},
						},
					},
				}, dst)
			},
			wantDst: &ArrayDst{
				A: []SimpleSrc{
					{
						Name:    "大明",
						Age:     ToPtr[int](18),
						Friends: []string{"Tom", "Jerry"},
					},
					{
						Name:    "小明",
						Age:     ToPtr[int](8),
						Friends: []string{"Tom"},
					},
				},
			},
		},
		{
			name: "成员为结构体数组，结构体不同",
			copyFunc: func() (any, error) {
				dst := &ArrayDst1{}
				return dst, CopyTo(&ArraySrc{
					A: []SimpleSrc{
						{
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
						{
							Name:    "小明",
							Age:     ToPtr[int](8),
							Friends: []string{"Tom"},
						},
					},
				}, dst)
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new([]SimpleSrc)).Elem(), reflect.TypeOf(new([]SimpleDst)).Elem(), "A"),
		},
		{
			name: "成员为map结构体",
			copyFunc: func() (any, error) {
				dst := &MapDst{}
				return dst, CopyTo(&MapSrc{
					A: map[string]SimpleSrc{
						"a": {
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
					},
				}, dst)
			},
			wantDst: &MapDst{
				A: map[string]SimpleSrc{
					"a": {
						Name:    "大明",
						Age:     ToPtr[int](18),
						Friends: []string{"Tom", "Jerry"},
					},
				},
			},
		},
		{
			name: "成员为不同的map结构体",
			copyFunc: func() (any, error) {
				dst := &MapDst1{}
				return dst, CopyTo(&MapSrc{
					A: map[string]SimpleSrc{
						"a": {
							Name:    "大明",
							Age:     ToPtr[int](18),
							Friends: []string{"Tom", "Jerry"},
						},
					},
				}, dst)
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new(map[string]SimpleSrc)).Elem(), reflect.TypeOf(new(map[string]SimpleDst)).Elem(), "A"),
		},
		{
			name: "成员有别名类型",
			copyFunc: func() (any, error) {
				dst := &SpecialDst1{}
				return dst, CopyTo(&SpecialSrc1{
					A: 1,
				}, dst)
			},
			wantErr: newErrTypeNotMatchError(reflect.TypeOf(new(int)).Elem(), reflect.TypeOf(new(aliasInt)).Elem(), "A"),
		},
		{
			name: "成员有别名类型1",
			copyFunc: func() (any, error) {
				dst := &SpecialDst2{}
				return dst, CopyTo(&SpecialSrc1{
					A: 1,
				}, dst)
			},
			wantDst: &SpecialDst2{A: 1},
		},
	}

	for _, tc := range testCases {
		t.Run(tc.name, func(t *testing.T) {
			res, err := tc.copyFunc()
			assert.Equal(t, tc.wantErr, err)
			if err != nil {
				return
			}
			assert.Equal(t, tc.wantDst, res)
		})
	}
}

func BenchmarkReflectCopier_Copy_PureRunTime(b *testing.B) {

	for i := 1; i <= b.N; i++ {
		_ = CopyTo(&SimpleSrc{
			Name:    "大明",
			Age:     ToPtr[int](18),
			Friends: []string{"Tom", "Jerry"},
		}, &SimpleDst{})
	}
}

func BenchmarkReflectCopier_CopyComplexStruct_WithPureRuntime(b *testing.B) {

	for i := 1; i <= b.N; i++ {
		_ = CopyTo(&ComplexSrc{
			Simple: SimpleSrc{
				Name:    "xiaohong",
				Age:     ToPtr[int](18),
				Friends: []string{"ha", "ha", "le"},
			},
			Embed: &EmbedSrc{
				SimpleSrc: SimpleSrc{
					Name:    "xiaopeng",
					Age:     ToPtr[int](88),
					Friends: []string{"la", "ha", "le"},
				},
				BasicSrc: &BasicSrc{
					Name:    "wang",
					Age:     22,
					CNumber: complex(2, 1),
				},
			},
			BasicSrc: BasicSrc{
				Name:    "wang11",
				Age:     22,
				CNumber: complex(2, 1),
			},
		}, &ComplexDst{})
	}
}
```