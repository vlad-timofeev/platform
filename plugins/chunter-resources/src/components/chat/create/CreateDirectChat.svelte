<!--
// Copyright © 2023 Hardcore Engineering Inc.
//
// Licensed under the Eclipse Public License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License. You may
// obtain a copy of the License at https://www.eclipse.org/legal/epl-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
//
// See the License for the specific language governing permissions and
// limitations under the License.
-->
<script lang="ts">
  import { createEventDispatcher, onMount } from 'svelte'
  import { deepEqual } from 'fast-equals'

  import { DirectMessage } from '@hcengineering/chunter'
  import contact, { Employee, PersonAccount } from '@hcengineering/contact'
  import core, { getCurrentAccount, Ref } from '@hcengineering/core'
  import { getResource } from '@hcengineering/platform'
  import { SelectUsersPopup } from '@hcengineering/contact-resources'
  import notification, { DocNotifyContext } from '@hcengineering/notification'
  import presentation, { createQuery, getClient } from '@hcengineering/presentation'
  import { Modal, showPopup } from '@hcengineering/ui'

  import chunter from '../../../plugin'
  import { buildDmName } from '../../../utils'
  import ChannelMembers from '../../ChannelMembers.svelte'

  const dispatch = createEventDispatcher()
  const client = getClient()
  const myAccId = getCurrentAccount()._id
  const query = createQuery()

  let employeeIds: Ref<Employee>[] = []
  let dmName = ''
  let accounts: PersonAccount[] = []
  let hidden = true

  $: loadDmName(accounts).then((r) => {
    dmName = r
  })
  $: query.query(contact.class.PersonAccount, { person: { $in: employeeIds } }, (res) => {
    accounts = res
  })

  async function loadDmName (employeeAccounts: PersonAccount[]) {
    return await buildDmName(client, employeeAccounts)
  }

  async function createDirectMessage () {
    const employeeAccounts = await client.findAll(contact.class.PersonAccount, { person: { $in: employeeIds } })
    const accIds = [myAccId, ...employeeAccounts.filter(({ _id }) => _id !== myAccId).map(({ _id }) => _id)].sort()

    const existingContexts = await client.findAll<DocNotifyContext>(
      notification.class.DocNotifyContext,
      {
        user: myAccId,
        attachedToClass: chunter.class.DirectMessage
      },
      { lookup: { attachedTo: chunter.class.DirectMessage } }
    )

    const navigate = await getResource(chunter.actionImpl.OpenChannel)

    for (const context of existingContexts) {
      if (deepEqual((context.$lookup?.attachedTo as DirectMessage)?.members.sort(), accIds)) {
        if (context.hidden) {
          await client.update(context, { hidden: false })
        }
        await navigate(context)

        return
      }
    }

    const dmId = await client.createDoc(chunter.class.DirectMessage, core.space.Space, {
      name: '',
      description: '',
      private: true,
      archived: false,
      members: accIds
    })

    const notifyContextId = await client.createDoc(notification.class.DocNotifyContext, core.space.Space, {
      user: myAccId,
      attachedTo: dmId,
      attachedToClass: chunter.class.DirectMessage,
      hidden: false
    })

    await navigate(undefined, undefined, { _id: notifyContextId, mode: 'direct' })
  }

  function handleCancel () {
    dispatch('close')
  }

  onMount(() => {
    openSelectUsersPopup(true)
  })

  function addMembersClicked () {
    openSelectUsersPopup(false)
  }

  function openSelectUsersPopup (closeOnClose: boolean) {
    showPopup(
      SelectUsersPopup,
      {
        okLabel: presentation.string.Next,
        skipCurrentAccount: true,
        selected: employeeIds
      },
      'top',
      (result?: Ref<Employee>[]) => {
        if (result != null) {
          employeeIds = result
          hidden = false
        } else if (closeOnClose) {
          dispatch('close')
        }
      }
    )
    hidden = true
  }
</script>

<Modal
  label={chunter.string.NewDirectChat}
  type="type-popup"
  {hidden}
  okLabel={presentation.string.Create}
  okAction={createDirectMessage}
  canSave={employeeIds.length > 0}
  onCancel={handleCancel}
  on:close
>
  <div class="hulyModal-content__titleGroup" style="padding: 0">
    <div class="title overflow-label mb-4" title={dmName}>
      {dmName}
    </div>

    <ChannelMembers
      ids={employeeIds}
      on:add={addMembersClicked}
      on:remove={(ev) => {
        employeeIds = employeeIds.filter((id) => id !== ev.detail)
      }}
    />
  </div>
</Modal>

<style lang="scss">
  .title {
    font-size: 1.25rem;
    font-weight: 500;
    padding: var(--spacing-1);
    max-width: 40rem;
    color: var(--global-primary-TextColor);
  }
</style>
