<template>
  <div class="card">
    <div class="card-header">
      <h5 class="card-title">
        {{ $route.meta.title }}
        <span v-if="savingOrder" class="text-muted ml-2" style="font-size:13px">
          <i class="fa fa-spinner fa-spin"></i> 儲存中…
        </span>
        <el-button @click="loadMenu" class="pull-right"><i class="fa fa-refresh"></i></el-button>
      </h5>
    </div>
    <div class="card-body">
      <!-- Toolbar -->
      <div class="d-flex mb-3" style="gap:8px;flex-wrap:wrap">
        <el-button @click="openCategoryDetail()">＋建立新類別</el-button>
        <el-button @click="openAddGroupModal()">＋建立新類別組別</el-button>
      </div>
      <div v-loading="loading">
        <div class="categories-layout">

          <!-- LEFT COLUMN: Menu order -->
          <div class="col-menu">
            <p class="section-label">
              <i class="fa fa-sort mr-1"></i> 菜單順序
              <small class="text-muted ml-2">（拖動組別 / 類別調整前台位置；從右側拖入即可加入選單）</small>
            </p>
            <draggable
              v-model="menuItems"
              handle=".drag-handle-top"
              filter=".no-drag-action"
              :group="{ name: 'menuTop', pull: true, put: ['catItems'] }"
              @change="onMenuItemsChange"
              :animation="150"
              class="top-level-list"
            >
              <div v-for="item in menuItems" :key="item._id" class="menu-row mb-2">
                <!-- GROUP -->
                <template v-if="item.type === 'group'">
                  <div class="group-block border rounded p-2" :class="{ 'is-collapsed': item._collapsed }">
                    <div class="group-header d-flex align-items-center mb-2" :class="{ 'mb-0': item._collapsed }">
                      <i class="fa fa-bars drag-handle-top mr-2"></i>
                      <i class="fa fa-folder-open text-warning mr-2"></i>
                      <template v-if="item._editing">
                        <el-input size="mini" v-model="item._editZh" placeholder="中文名稱" style="width:120px" class="mr-1"></el-input>
                        <el-input size="mini" v-model="item._editEn" placeholder="English name" style="width:120px" class="mr-1"></el-input>
                        <el-button size="mini" type="primary" @click="saveGroupName(item)">儲存</el-button>
                        <el-button size="mini" @click="$set(item, '_editing', false)">取消</el-button>
                      </template>
                      <template v-else>
                        <strong class="mr-auto">{{ item.name.zh }} / {{ item.name.en }}<span class="group-count">（{{ item.categories ? item.categories.length : 0 }}）</span></strong>
                        <el-switch v-model="item.publish" @change="changeGroupPublish(item.id || item._id, $event)" active-color="#13ce66" inactive-color="#ff4949" class="mr-2"></el-switch>
                        <span class="mr-2 publish-label">{{ item.publish ? '已發佈' : '未發佈' }}</span>
                        <el-button size="mini" class="no-drag-action group-collapse-btn" @mousedown.native.stop @touchstart.native.stop @click.stop.prevent="toggleGroupCollapse(item)">
                          <i :class="item._collapsed ? 'fa fa-chevron-down' : 'fa fa-chevron-up'"></i>
                        </el-button>
                        <el-button size="mini" icon="el-icon-edit" class="no-drag-action" @mousedown.native.stop @touchstart.native.stop @click.stop="startEditGroup(item)"></el-button>
                        <el-button size="mini" type="danger" icon="el-icon-delete" class="ml-1 no-drag-action" @mousedown.native.stop @touchstart.native.stop @click.stop.prevent="deleteGroup(item)"></el-button>
                      </template>
                    </div>
                    <draggable
                      v-if="!item._collapsed"
                      v-model="item.categories"
                      handle=".drag-handle-inner"
                      :group="{ name: 'catItems', pull: true, put: true }"
                      @change="onGroupCategoriesChange($event, item)"
                      :animation="150"
                      class="inner-list"
                    >
                      <div
                        v-for="cat in item.categories"
                        :key="cat._id || cat.id"
                        class="category-row d-flex align-items-center py-1 px-2 rounded mb-1"
                      >
                        <i class="fa fa-bars drag-handle-inner mr-2"></i>
                        <img v-if="cat.images && cat.images[0]" :src="cat.images[0].small.path" width="40" height="40" alt="" class="mr-2 cat-thumb">
                        <span v-else class="mr-2 no-img-placeholder">無圖</span>
                        <span class="mr-auto cat-name">{{ cat.name.zh }}</span>
                        <el-switch v-model="cat.publish" @change="changePublish(cat.id || cat._id, $event)" active-color="#13ce66" inactive-color="#ff4949" class="mr-2"></el-switch>
                        <span class="mr-2 publish-label">{{ cat.publish ? '已發佈' : '未發佈' }}</span>
                        <p-button type="success" size="sm" icon @click.stop="openCategoryDetail(cat.id || cat._id)">
                          <i class="fa fa-edit"></i>
                        </p-button>
                      </div>
                      <div v-if="!item.categories || item.categories.length === 0" class="py-2 pl-2 drop-hint">
                        （將類別拖到這裡加入此組別）
                      </div>
                    </draggable>
                  </div>
                </template>
                <!-- UNGROUPED CATEGORY (menu position row) -->
                <template v-else-if="item.type === 'category'">
                  <div class="category-row ungrouped d-flex align-items-center py-1 px-2 rounded border">
                    <i class="fa fa-bars drag-handle-top mr-2"></i>
                    <img v-if="item.images && item.images[0]" :src="item.images[0].small.path" width="40" height="40" alt="" class="mr-2 cat-thumb">
                    <span v-else class="mr-2 no-img-placeholder">無圖</span>
                    <span class="mr-auto cat-name">{{ item.name.zh }}</span>
                    <span class="text-muted mr-3" style="font-size:11px">未分組</span>
                    <el-switch v-model="item.publish" @change="changePublish(item.id || item._id, $event)" active-color="#13ce66" inactive-color="#ff4949" class="mr-2"></el-switch>
                    <span class="mr-2 publish-label">{{ item.publish ? '已發佈' : '未發佈' }}</span>
                    <p-button type="success" size="sm" icon @click.stop="openCategoryDetail(item.id || item._id)">
                      <i class="fa fa-edit"></i>
                    </p-button>
                  </div>
                </template>
              </div>
              <div v-if="!loading && menuItems.length === 0" class="py-4 drop-hint text-center">
                （菜單為空 — 從右側拖入類別或組別）
              </div>
            </draggable>
          </div>

          <!-- RIGHT COLUMN: Ungrouped pool
               Drag FROM here INTO left menu list → adds as standalone menu item
               Drag FROM here INTO a group        → assigns to group
               Drag FROM a group back HERE        → removes from group
               Drag FROM menu list back HERE      → removes from menu order -->
          <div class="col-pool">
            <p class="section-label">
              <i class="fa fa-th-large mr-1"></i> 未分組類別
              <small class="text-muted ml-2">（拖入左側選單加入；拖入組別分組；從組別或選單拖回此處取消）</small>
            </p>
            <draggable
              v-model="ungroupedPool"
              handle=".drag-handle-inner"
              :group="{ name: 'catItems', pull: ['catItems', 'menuTop'], put: true }"
              @change="onUngroupedPoolChange"
              :animation="150"
              class="ungrouped-pool"
            >
              <div
                v-for="cat in ungroupedPool"
                :key="cat._id || cat.id"
                class="category-row d-flex align-items-center py-1 px-2 rounded mb-1 border"
              >
                <i class="fa fa-bars drag-handle-inner mr-2"></i>
                <img v-if="cat.images && cat.images[0]" :src="cat.images[0].small.path" width="40" height="40" alt="" class="mr-2 cat-thumb">
                <span v-else class="mr-2 no-img-placeholder">無圖</span>
                <span class="mr-auto cat-name">{{ cat.name.zh }}</span>
                <el-switch v-model="cat.publish" @change="changePublish(cat.id || cat._id, $event)" active-color="#13ce66" inactive-color="#ff4949" class="mr-2"></el-switch>
                <span class="mr-2 publish-label">{{ cat.publish ? '已發佈' : '未發佈' }}</span>
                <p-button type="success" size="sm" icon @click.stop="openCategoryDetail(cat.id || cat._id)">
                  <i class="fa fa-edit"></i>
                </p-button>
              </div>
              <div v-if="ungroupedPool.length === 0" class="py-3 drop-hint text-center">
                （沒有未分組類別 — 從組別或選單拖出放在這裡）
              </div>
            </draggable>
          </div>

        </div>
      </div>
    </div>
    <CategoryDetail ref="CategoryDetail"></CategoryDetail>
    <AddCategoryGroupModal ref="AddCategoryGroupModal"></AddCategoryGroupModal>
    <ConfirmDialog ref="ConfirmDialog"></ConfirmDialog>
  </div>
</template>
<script>
import {Button, Select, Option, Switch, Loading, Input} from 'element-ui'
import draggable                                        from 'vuedraggable'
import CategoryDetail                                   from './components/CategoryDetail'
import AddCategoryGroupModal                            from './components/AddCategoryGroupModal'
import ConfirmDialog                                    from '@/components/Dashboard/Views/Components/ConfirmDialog'

export default {
  name       : 'Categories',
  components : {
    [Select.name] : Select,
    [Option.name] : Option,
    [Button.name] : Button,
    [Switch.name] : Switch,
    [Input.name]  : Input,
    draggable,
    CategoryDetail,
    AddCategoryGroupModal,
    ConfirmDialog,
  },
  directives : {
    loading : Loading.directive,
  },
  data () {
    return {
      loading            : false,
      savingOrder        : false,
      menuItems          : [],   // full ordered list (groups + ungrouped) — drives top-level order drag
      ungroupedPool      : [],   // ungrouped categories only — drives cross-list assignment drag
      collapsedGroupsKey : 'yunerskin-admin:categories:collapsed-groups',
      collapsedGroupState : {},
    }
  },
  created () {
    this.collapsedGroupState = this.readCollapsedGroupState()
    this.loadMenu()
  },
  methods : {
    getGroupId (item) {
      return String(item.id || item._id)
    },
    readCollapsedGroupState () {
      if (typeof window === 'undefined' || !window.localStorage) return {}
      try {
        const raw = window.localStorage.getItem(this.collapsedGroupsKey)
        return raw ? JSON.parse(raw) : {}
      } catch (e) {
        return {}
      }
    },
    saveCollapsedGroupState () {
      if (typeof window === 'undefined' || !window.localStorage) return
      try {
        window.localStorage.setItem(this.collapsedGroupsKey, JSON.stringify(this.collapsedGroupState))
      } catch (e) {
        // Ignore storage failures; the UI still works during this session.
      }
    },
    syncCollapsedGroupState (groups) {
      const nextState = {}
      groups.forEach(group => {
        const groupId = this.getGroupId(group)
        nextState[groupId] = !!this.collapsedGroupState[groupId]
      })
      this.collapsedGroupState = nextState
      this.saveCollapsedGroupState()
    },
    /* ─── Data loading ────────────────────────────────────────── */
    async loadMenu () {
      this.loading = true
      try {
        // Load menu first so loadUngroupedPool can exclude already-in-menu IDs
        const menuRes = await this.$api.get('/v1/categories/menu/admin')
        if (menuRes && menuRes.data) {
          this.menuItems = menuRes.data.map(item => {
            const isGroup = item.type === 'group'
            const groupId = this.getGroupId(item)
            const collapsed = isGroup ? !!this.collapsedGroupState[groupId] : false

            return {
              ...item,
              _editing   : false,
              _collapsed : collapsed,
              _editZh    : item.name ? item.name.zh : '',
              _editEn    : item.name ? item.name.en : '',
            }
          })
          this.syncCollapsedGroupState(this.menuItems.filter(item => item.type === 'group'))
        }
        await this.loadUngroupedPool()
      } catch (e) {
        console.error(e)
        this.$notify.error({ title : '錯誤', message : '載入菜單失敗' })
      } finally {
        this.loading = false
      }
    },

    // Fetch ALL categories from the flat-list API and keep only those without a group
    // AND not already present as a standalone entry in the menu order.
    async loadUngroupedPool () {
      try {
        const res = await this.$api.get('/v1/categories', {
          params : { limit : 500, page : 1 },
        })
        if (res && res.data && res.data.results) {
          // Build a set of IDs already in the top-level menu as type:"category".
          // Use id || _id to handle both Mongoose virtual and raw ObjectId field names.
          const inMenuIds = new Set(
            this.menuItems
                .filter(item => item.type === 'category')
                .map(item => String(item.id || item._id)),
          )
          this.ungroupedPool = res.data.results
                                  .filter(cat => !cat.group && !inMenuIds.has(String(cat.id || cat._id)))
                                  .map(cat => ({ ...cat }))
        }
      } catch (e) {
        console.error('Failed to load ungrouped pool', e)
      }
    },
    /* ─── Category edit / create ─────────────────────────────── */
    async openCategoryDetail (categoryId) {
      const result = await this.$refs.CategoryDetail.open(categoryId)
      if (result) {
        this.loadMenu()
      }
    },
    /* ─── Group management ───────────────────────────────────── */
    async openAddGroupModal () {
      const result = await this.$refs.AddCategoryGroupModal.open()
      if (result) {
        this.$notify({ title : '成功', message : '類別組別已建立', type : 'success' })
        this.loadMenu()
      }
    },
    startEditGroup (item) {
      this.$set(item, '_editZh', item.name.zh)
      this.$set(item, '_editEn', item.name.en)
      this.$set(item, '_editing', true)
    },
    toggleGroupCollapse (item) {
      const nextCollapsed = !item._collapsed
      const groupId = this.getGroupId(item)
      this.$set(item, '_collapsed', nextCollapsed)
      this.$set(this.collapsedGroupState, groupId, nextCollapsed)
      this.saveCollapsedGroupState()
    },
    async saveGroupName (item) {
      try {
        await this.$api.patch('/v1/categories/groups/' + item._id, {
          name : { zh : item._editZh, en : item._editEn },
        })
        item.name.zh = item._editZh
        item.name.en = item._editEn
        this.$set(item, '_editing', false)
        this.$notify({ title : '成功', message : '組別名稱已更新', type : 'success' })
      } catch (e) {
        this.$notify.error({ title : '錯誤', message : '更新組別名稱失敗' })
      }
    },
    async changeGroupPublish (groupId, val) {
      const loading = new Loading.service({ fullscreen : false })
      try {
        await this.$api.patch('/v1/categories/groups/' + groupId, { publish : val })
        this.$notify({ title : '成功', message : '更新成功', type : 'success' })
      } catch (e) {
        this.$notify.error({ title : '錯誤', message : '更新失敗' })
      } finally {
        loading.close()
      }
    },
    async deleteGroup (item) {
      const groupId = item.id || item._id
      try {
        const confirmation = await this.$refs.ConfirmDialog.open(
          '確認刪除',
          `刪除「${ item.name.zh }」組別？組內類別將變為未分組類別。`,
        )
        if (!confirmation.isConfirmed) return
      } catch (e) {
        return
      }
      try {
        const groupCategoryIds = (item.categories || [])
        .map(cat => cat.id || cat._id)
        .filter(Boolean)
        if (groupCategoryIds.length) {
          await Promise.all(
            groupCategoryIds.map(catId =>
              this.$api.patch('/v1/categories/' + catId, { group : null }),
            ),
          )
        }
        await this.$api.delete('/v1/categories/groups/' + groupId)
        this.$delete(this.collapsedGroupState, String(groupId))
        this.saveCollapsedGroupState()
        this.$notify({ title : '成功', message : '組別已刪除', type : 'success' })
        await this.loadMenu()
      } catch (e) {
        this.$notify.error({ title : '錯誤', message : '刪除組別失敗' })
      }
    },
    /* ─── Top-level menu order ────────────────────────────────── */
    // Pure API call — callers are responsible for savingOrder flag
    async saveMenuOrder () {
      const items = this.menuItems.map(item => ({
        type  : item.type,
        refId : item.id || item._id,
      }))
      await this.$api.put('/v1/categories/menu-order', { items })
    },
    /* ─── Top-level draggable @change ─────────────────────────── */
    // event.moved  → same-list reorder → save new menu order
    // event.added  → category dragged in from ungroupedPool → tag as type:"category", save, reload
    // event.removed→ item dragged OUT to ungroupedPool; handled by onUngroupedPoolChange
    async onMenuItemsChange (event) {
      if (event.moved) {
        this.savingOrder = true
        try {
          await this.saveMenuOrder()
        } catch (e) {
          this.$notify.error({ title : '錯誤', message : '儲存排序失敗' })
        } finally {
          this.savingOrder = false
        }
      } else if (event.added) {
        const item = event.added.element
        // Inject type so the template renders the correct row variant
        this.savingOrder = true
        try {
          const itemId = item.id || item._id
          if (item.type === 'group') {
            await this.deleteGroup(item)
            return
          }
          if (item.group) {
            await this.$api.patch('/v1/categories/' + itemId, { group : null })
            this.$set(item, 'group', null)
          }
          this.$set(item, 'type', 'category')
          await this.saveMenuOrder()
          this.$notify({ title : '成功', message : `「${ item.name.zh }」已加入菜單`, type : 'success' })
          await this.loadMenu()
        } catch (e) {
          this.$notify.error({ title : '錯誤', message : '加入菜單失敗' })
          await this.loadMenu()
        } finally {
          this.savingOrder = false
        }
      }
    },
    /* ─── Group inner list @change ────────────────────────────── */
    // event.moved  → same-list reorder within this group
    // event.added  → category dragged IN from another group or from the ungrouped pool
    // event.removed→ handled by the destination list's "added" event
    async onGroupCategoriesChange (event, group) {
      if (event.moved) {
        await this._patchGroupSortOrders(group)
      } else if (event.added) {
        const cat = event.added.element
        const catId = cat.id || cat._id
        this.savingOrder = true
        try {
          const menuIndex = this.menuItems.findIndex(item => String(item.id || item._id) === String(catId))
          if (menuIndex !== -1) {
            this.menuItems.splice(menuIndex, 1)
          }
          await this.$nextTick()
          await this.saveMenuOrder()
          await Promise.all(
            group.categories.map((c, index) =>
              this.$api.patch('/v1/categories/' + (c.id || c._id), {
                group     : group._id,
                sortOrder : index,
              }),
            ),
          )
          this.$notify({ title : '成功', message : `「${ cat.name.zh }」已移入「${ group.name.zh }」`, type : 'success' })
          await this.loadMenu()
        } catch (e) {
          this.$notify.error({ title : '錯誤', message : '移動類別失敗' })
          await this.loadMenu()
        } finally {
          this.savingOrder = false
        }
      } else if (event.removed) {
        await this._patchGroupSortOrders(group)
      }
    },
    /* ─── Ungrouped pool @change ──────────────────────────────── */
    // event.added → category dragged in from:
    //   a) a group inner list → ungroup it via PATCH, reload
    //   b) menuItems (type:"category") → it's already ungrouped; just save menu order, reload
    // event.moved → reorder within pool (no API call needed)
    async onUngroupedPoolChange (event) {
      if (event.added) {
        const cat = event.added.element
        const catId = cat.id || cat._id
        const fromGroup = !!cat.group   // cat.group is set when coming from a group inner-list
        const isGroup = cat.type === 'group'
        this.savingOrder = true
        try {
          if (isGroup) {
            const confirmation = await this.$refs.ConfirmDialog.open(
              '確認刪除',
              `刪除「${ cat.name.zh }」組別？組內類別將變為未分組類別。`,
            )
            if (!confirmation.isConfirmed) {
              await this.loadMenu()
              return
            }
            // Whole group dropped into the pool: delete the group so its categories
            // become ungrouped again via the backend.
            const groupCategoryIds = (cat.categories || [])
            .map(child => child.id || child._id)
            .filter(Boolean)
            if (groupCategoryIds.length) {
              await Promise.all(
                groupCategoryIds.map(childId =>
                  this.$api.patch('/v1/categories/' + childId, { group : null }),
                ),
              )
            }
            await this.$api.delete('/v1/categories/groups/' + catId)
            this.$notify({ title : '成功', message : `「${ cat.name.zh }」組別已刪除`, type : 'success' })
          } else if (fromGroup) {
            // Dragged out of a group → clear the group assignment
            await this.$api.patch('/v1/categories/' + catId, { group : null })
            this.$notify({ title : '成功', message : `「${ cat.name.zh }」已取消分組`, type : 'success' })
          } else {
            // Dragged back from a top-level menuItems row (type:"category")
            // Persist menu order here after the destination list has settled.
            await this.$nextTick()
            await this.saveMenuOrder()
            this.$notify({ title : '成功', message : `「${ cat.name.zh }」已從菜單移除`, type : 'success' })
          }
          await this.loadMenu()
        } catch (e) {
          this.$notify.error({ title : '錯誤', message : fromGroup ? '取消分組失敗' : '更新菜單失敗' })
          await this.loadMenu()
        } finally {
          this.savingOrder = false
        }
      }
    },
    /* ─── Internal: patch sortOrder for all cats in a group ──── */
    async _patchGroupSortOrders (group) {
      this.savingOrder = true
      try {
        await Promise.all(
          group.categories.map((cat, index) =>
            this.$api.patch('/v1/categories/' + (cat.id || cat._id), { sortOrder : index }),
          ),
        )
      } catch (e) {
        this.$notify.error({ title : '錯誤', message : '儲存組內排序失敗' })
      } finally {
        this.savingOrder = false
      }
    },
    /* ─── Publish toggle ─────────────────────────────────────── */
    async changePublish (categoryId, val) {
      const loading = new Loading.service({ fullscreen : false })
      try {
        await this.$api.patch('/v1/categories/' + categoryId, { publish : val })
        this.$notify({ title : '成功', message : '更新成功', type : 'success' })
      } catch (e) {
        this.$notify.error({ title : '錯誤', message : '更新失敗' })
      } finally {
        loading.close()
      }
    },
  },
}
</script>
<style scoped>
.section-label {
  font-weight: 600;
  font-size: 13px;
  color: #555;
  margin-bottom: 8px;
  margin-top: 4px;
}

.drag-handle-top,
.drag-handle-inner {
  cursor: grab;
  color: #bbb;
  font-size: 14px;
  flex-shrink: 0;
}

.drag-handle-top:active,
.drag-handle-inner:active {
  cursor: grabbing;
}

.group-block {
  border-color: #ddd !important;
  background: #fafafa;
}

.group-block.is-collapsed {
  background: #f7f7f7;
}

.group-block.is-collapsed .group-header {
  margin-bottom: 0 !important;
}

.group-block.is-collapsed .inner-list {
  display: none;
}

.group-count {
  font-weight: 400;
  color: #888;
  font-size: 12px;
}

.group-collapse-btn {
  min-width: 32px;
  padding-left: 0.45rem;
  padding-right: 0.45rem;
}

.inner-list {
  padding-left: 28px;
  min-height: 40px;
}


.top-level-list {
  min-height: 20px;
}

.ungrouped-pool {
  min-height: 54px;
  max-height: calc(100vh - 210px);
  overflow-y: auto;
  border: 2px dashed #d0d0d0;
  border-radius: 6px;
  padding: 8px;
}

.category-row {
  background: #fff;
  border-color: #eee !important;
  user-select: none;
}

.cat-thumb {
  object-fit: cover;
  border-radius: 4px;
  flex-shrink: 0;
}

.no-img-placeholder {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 10px;
  color: #999;
  flex-shrink: 0;
}

.cat-name {
  font-size: 14px;
}

.publish-label {
  font-size: 12px;
  min-width: 42px;
  text-align: right;
}

.drop-hint {
  font-size: 13px;
  border: 1px dashed #ccc;
  border-radius: 4px;
  color: #aaa;
}

/* ── Two-column layout ─────────────────────────────────────── */
.categories-layout {
  display: flex;
  gap: 16px;
  align-items: flex-start;
}

.col-menu {
  flex: 1 1 55%;
  min-width: 0;
}

.col-pool {
  flex: 0 0 340px;
  width: 340px;
  position: sticky;
  top: 80px;
  max-height: calc(100vh - 120px);
  overflow-y: auto;
}

/* Mobile: stack vertically */
@media (max-width: 768px) {
  .categories-layout {
    flex-direction: column;
  }

  .col-pool {
    position: static;
    flex: 1 1 100%;
    width: 100%;
    max-height: none;
  }

  .ungrouped-pool {
    max-height: none;
  }
}
</style>
