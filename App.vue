<template>
  <tree-node :data="treeData" />
</template>
<script>
import TreeNode from './TreeNode'

export default {
  components: { TreeNode },
  data: () => ({
    treeData: {
      label: 'Company',
      children: [
        { label: 'Ken',
          children: [
            { label: 'Ben' },
            { label: 'Karen',
              children: [
                { label: 'Amy' }
              ]
            },
            { label: 'John' },
            { label: 'Mike',
              children: [
                { label: 'Mary',
                  children: [
                    { label: 'James',
                      children: [
                        { label: 'Susan' }
                      ]
                    }
                  ]
                },
                { label: 'Lisa' }
              ]
            }
          ]
        },
        {
          label: 'Jason',
          children: [
            { label: 'Rose' }
          ]
        }
      ]
    }
  })
}
</script>
