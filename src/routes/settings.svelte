<script>
  //Vendors
  import { navigate, Link } from "svelte-routing";
  import { onMount } from "svelte";

  import SocialShare from "../modules/share/share";
  import Storage from "../modules/storage/storage";
  // Components
  import NItem from "../components/list-item/list-item.svelte";
  import NText from "../components/text/text.svelte";
  import NToggle from "../components/toggle-switch/toggle-switch.svelte";
  import NToolbar from "../components/toolbar/toolbar.svelte";

  import BlockstackOptions from "../components/storage/blockstack.svelte";
  import LocalstorageOptions from "../components/storage/localstorage.svelte";
  import PouchDBOptions from "../components/storage/pouchdb.svelte";

  // Containers
  import StorageManager from "../containers/storage/storage.svelte";
  import ImporterModal from "../containers/importer/importer.svelte";
  import MassEditor from "../containers/mass-editor/mass-editor.svelte";

  import AppLayout from "../containers/layout/app.svelte";

  // Vendors
  import dayjs from "dayjs";
  import localforage from "localforage";

  // Stores
  import { UserStore } from "../store/user";
  import { LedgerStore } from "../store/ledger";
  import { Interact } from "../store/interact";
  import { TrackerStore } from "../store/trackers";
  import { BoardStore } from "../store/boards";
  import { NomieAPI } from "../store/napi";
  import { Lang } from "../store/lang";

  // Config
  import config from "../../config/global";

  // consts
  // const Export = new Exporter();

  let data = {
    signedIn: false,
    files: [],
    showMassEditor: false
  };

  $: alwaysLocate = $UserStore.alwaysLocate;

  let ledger = null;
  let trackers = null;
  // let user = null;
  let fileInput;
  let showImporter = false;

  let methods = {
    sign_out() {
      UserStore.signout();
    },
    share() {
      SocialShare(
        `I track my life with Nomie! It's free, private, and you get to design what you track. @nomieapp`,
        "https://nomie.app"
      );
    },
    sign_in() {
      UserStore.redirectToSignIn();
    },
    closeMassEditor() {
      data.showMassEditor = false;
    },
    bookAge(date) {
      return dayjs(`${date}-01`).fromNow();
    },
    faq() {
      navigate("/faq");
    },

    // switchToCloud() {
    //   let msg = Lang.t("settings.switch-to-cloud-notice");
    //   Interact.confirm(Lang.t("settings.switch-to-cloud-confirm"), msg).then(
    //     res => {
    //       if (res === true) {
    //         UserStore.setStorage("blockstack");
    //         window.location.href = "/";
    //       }
    //     }
    //   );
    // },
    // switchToLocal() {
    //   let msg = Lang.t("settings.switch-to-local-notice");
    //   Interact.confirm(Lang.t("settings.switch-to-local-confirm"), msg).then(
    //     res => {
    //       if (res === true) {
    //         UserStore.setStorage("local");
    //         window.location.href = "/";
    //       }
    //     }
    //   );
    // },
    settingChange() {
      UserStore.saveMeta();
    },
    async deleteEverything() {
      try {
        let res = await Interact.confirm(
          "DANGER ZONE!",
          `This will destroy all of your data in Nomie. Are you sure?`,
          "Destroy"
        );
        if (res) {
          res = await Interact.confirm(
            "Sorry! One last time.. Really?",
            `You will basically be starting over from scratch... You good with that?`,
            "Destroy!"
          );

          if (res === true) {
            let files = await Storage.list();

            let promises = [];
            files.forEach(file => {
              promises.push(Storage.delete(file));
            });
            await Promise.all(promises);
            await localforage.clear();
            localStorage.clear();
            await Interact.alert("Done", "Your data has been destroyed.");

            console.log("redirecting");
            window.location.href = "/";
          }
        } // end if confirmed
      } catch (e) {
        Interact.alert(Lang.t("general.error"), e.message);
      }
    },
    switchStorage(type) {
      let from = $UserStore.storageType;
      let to = type;
      let conf = confirm(
        `${Lang.t(
          "storage.switch",
          `Switch from ${from} to ${to}? You can always switch back. 

Note: Your data will not automatically move over. You'll first need to export it, then you can import into this new storage.`
        )}`
      );
      if (conf === true) {
        if (["local", "pouchdb", "blockstack"].indexOf(to) > -1) {
          UserStore.setStorage(to);
          Interact.reload();
        } else {
          alert(`Error: ${to} is not valid`);
        }
      }
    },
    storageMenu() {
      let buttons = [
        {
          title: `${$UserStore.storageType === "local" ? "✓" : ""} ${Lang.t(
            "storage.local_title",
            "This Device Only"
          )}`,
          description: Lang.t(
            "storage.local_description",
            "Good for getting started, but make sure you backup your data."
          ),
          click() {
            methods.switchStorage("local");
          }
        },
        {
          title: `${
            $UserStore.storageType === "blockstack" ? "✓" : ""
          } Blockstack`,
          description: Lang.t(
            "storage.blockstack_description",
            "Stored encryted with Blockstack's file storage. FREE"
          ),
          click() {
            methods.switchStorage("blockstack");
          }
        },
        {
          title: `${$UserStore.storageType === "pouchdb" ? "✓" : ""} ${Lang.t(
            "storage.pouchdb_title",
            "Device + My own CouchDB"
          )}`,
          description: Lang.t(
            "storage.pouchdb_description",
            "Have a CouchDB server? Sync in near-real time."
          ),
          click() {
            methods.switchStorage("pouchdb");
          }
        }
      ];
      Interact.popmenu({
        title: Lang.t("storage.type_selector_title", `Storage Options`),
        description: Lang.t(
          "storage.type_selector_title",
          `How would you like your data stored?`
        ),
        buttons: buttons
      });
    },
    // boardsToggle() {
    //   $UserStore.meta.boardsEnabled = !$UserStore.meta.boardsEnabled;
    //   UserStore.saveMeta();
    // },
    lockToggle() {
      if ($UserStore.meta.lock === true) {
        if (($UserStore.meta.pin || "").length == 0) {
          // TODO: figure out how to handle a cancel in the interact prompt
          Interact.prompt(Lang.t("settings.pin-details"), null, {
            value: "",
            valueType: "number"
          }).then(pin => {
            if (!pin) {
              $UserStore.meta.lock = false;
              $UserStore.meta.pin = null;
              UserStore.saveMeta();
            } else {
              $UserStore.meta.lock = true;
              $UserStore.meta.pin = pin;
              UserStore.saveMeta();
            }
          });
        }
      } else {
        $UserStore.meta.lock = false;
        $UserStore.meta.pin = null;
        UserStore.saveMeta();
      }
    }
  };

  // const setTimeout = setTimeout;
  onMount(() => {
    window.scrollTo(0, 0);
  });
</script>

<AppLayout>
  <div slot="header">
    <NToolbar>
      <div class="text-inverse text-bold">{Lang.t('settings.settings')}</div>
      <button on:click={methods.faq} class="btn btn-clear text-primary">
        {Lang.t('general.faq')}
      </button>
    </NToolbar>
  </div>

  <div slot="content">
    {#if $UserStore.meta}
      <div class="page page-settings">
        <div class="container p-0 n-list">

          <NItem className="n-item-divider compact" />
          <NItem
            className="clickable pt-2"
            title={Lang.t('settings.share-nomie', 'Share Nomie w/ Friends')}
            description="It'd be super helpful"
            on:click={methods.share}>
            <span slot="left" class="btn-icon zmdi text-primary zmdi-share" />
            <span slot="right" class="icon zmdi zmdi-more" />
          </NItem>

          <div class="n-pop">
            <NItem
              title={Lang.t('general.customize')}
              className="n-item-divider" />
            <NItem title={Lang.t('settings.theme')}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-invert-colors" />
              <div slot="right">
                <select
                  class="form-control"
                  style="min-width:100px;width:100px"
                  bind:value={$UserStore.theme}
                  on:change={event => {
                    UserStore.setTheme($UserStore.theme);
                  }}>
                  <option value="auto">Auto</option>
                  <option value="dark">Dark</option>
                  <option value="light">Light</option>
                </select>

              </div>
            </NItem>
            <!-- Use Location -->
            <NItem title={Lang.t('settings.use-location')}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-my-location" />
              <div slot="right">
                <NToggle
                  bind:value={$UserStore.alwaysLocate}
                  on:change={event => {
                    UserStore.setAlwaysLocate(event.detail);
                  }} />
              </div>
            </NItem>
            <!-- Tracker Board Tabs -->
            {#if $BoardStore.boards.length == 0}
              <NItem title={Lang.t('settings.enable-boards')}>
                <span slot="left" class="btn-icon zmdi text-primary zmdi-tab" />
                <div slot="right">
                  <NToggle
                    bind:value={$UserStore.meta.boardsEnabled}
                    on:change={methods.settingChange} />
                </div>
              </NItem>
            {:else}
              <NItem
                title={Lang.t('settings.enable-boards')}
                className="disabled">
                <span slot="left" class="btn-icon zmdi text-primary zmdi-tab" />
                <div slot="right">
                  <NToggle value={true} locked={true} />
                </div>
              </NItem>
            {/if}
            <!-- Pin Code -->
            <NItem title={Lang.t('settings.require-pin')}>
              <span slot="left" class="btn-icon zmdi text-primary zmdi-apps" />
              <div slot="right">
                <NToggle
                  bind:value={$UserStore.meta.lock}
                  on:change={methods.lockToggle} />
              </div>
            </NItem>
            <!-- 24 Hour -->
            <NItem title={Lang.t('settings.24-hour-clock')}>
              <span slot="left" class="btn-icon zmdi text-primary zmdi-time" />
              <div slot="right">
                <NToggle
                  bind:value={$UserStore.meta.is24Hour}
                  on:change={methods.settingChange} />
              </div>
            </NItem>
            <!-- Language -->
            <NItem title={Lang.t('settings.language')}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-translate" />
              <div slot="right">
                <select
                  class="form-control"
                  style="min-width:100px;width:100px"
                  bind:value={$Lang.lang}
                  on:change={event => {
                    Lang.setLang($Lang.lang);
                  }}>
                  {#each Lang.getLangs() as lang}
                    <option value={lang.key}>{lang.label}</option>
                  {/each}
                </select>

              </div>
            </NItem>

          </div>

          <div class="n-pop">
            <NItem title={Lang.t('settings.data')} className="n-item-divider" />
            <NItem
              className="clickable"
              title={Lang.t('settings.nomie-api')}
              on:click={() => navigate('/api')}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-code-setting" />
              <span slot="right" class="icon zmdi zmdi-chevron-right" />
            </NItem>
            <NItem
              className="clickable"
              title={Lang.t('settings.import-from-backup')}
              on:click={() => {
                showImporter = true;
              }}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-cloud-download" />
              <span slot="right" class="icon zmdi zmdi-chevron-right" />
              <input
                slot="right"
                class="d-none"
                type="file"
                bind:this={fileInput}
                on:change={methods.onImportFile} />
            </NItem>

            <NItem
              className="clickable"
              title={Lang.t('settings.generate-backup')}
              to="/settings/export/backup">
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-cloud-upload" />
              <span slot="right" class="icon zmdi zmdi-chevron-right" />
            </NItem>
            <NItem
              className="clickable"
              title={Lang.t('settings.generate-csv')}
              to="/settings/export/csv">
              <span slot="left" class="btn-icon zmdi text-primary zmdi-grid" />
              <span slot="right" class="icon zmdi zmdi-chevron-right" />
            </NItem>
            <NItem
              className="clickable"
              title="{Lang.t('settings.find-and-replace')}..."
              on:click={() => {
                data.showMassEditor = true;
              }}>
              <span
                slot="left"
                class="btn-icon zmdi text-primary zmdi-search-replace" />
              <span slot="right" class="icon zmdi zmdi-chevron-right" />
            </NItem>

            <MassEditor
              on:close={methods.closeMassEditor}
              show={data.showMassEditor} />

          </div>
          <NItem
            title={Lang.t('settings.storage')}
            className="n-item-divider" />

          <NItem>
            <div class="title truncate">
              <strong>{Lang.t('general.type', 'Type')}</strong>
            </div>
            <span slot="left" class="btn-icon zmdi text-primary zmdi-storage" />

            <div slot="right">

              <button
                class="btn btn-clear icon-right"
                on:click={methods.storageMenu}>
                {#if $UserStore.storageType === 'local'}
                  {Lang.t('storage.local', 'Local')}
                {:else if $UserStore.storageType === 'pouchdb'}
                  {Lang.t('storage.pouchdb', 'Local + CouchDB')}
                {:else if $UserStore.storageType === 'blockstack'}
                  {Lang.t('storage.blockstack', 'Blockstack')}
                {/if}
                <i class="zmdi zmdi-chevron-down ml-2" />
              </button>
              <!-- {#if $UserStore.storageType === 'local'}
                <button
                  class="btn btn-clear text-primary"
                  on:click={methods.switchToCloud}>
                  {Lang.t('settings.use-cloud')}
                </button>
              {:else}
                <button
                  class="btn btn-clear text-primary"
                  on:click={methods.switchToLocal}>
                  {Lang.t('settings.use-local')}
                </button>
              {/if} -->
            </div>
          </NItem>

          {#if $UserStore.storageType === 'blockstack'}
            <BlockstackOptions />
          {/if}
          {#if $UserStore.storageType === 'local'}
            <LocalstorageOptions />
          {/if}
          {#if $UserStore.storageType === 'pouchdb'}
            <PouchDBOptions />
          {/if}

          <NItem title="DB Stats" className="n-item-divider" />
          <NItem title={Lang.t('general.trackers', 'Trackers')}>
            <span slot="right">{TrackerStore.getAsArray().length}</span>
          </NItem>
          <StorageManager />

          <div class="n-pop">
            <NItem
              title={Lang.t('settings.about-nomie')}
              className="n-item-divider" />
            <NItem title="Learn More">
              <span slot="right">
                <a
                  href="https://nomie.app?s=dap"
                  class="btn btn-clear text-primary"
                  target="_system">
                  Website
                </a>
              </span>
            </NItem>
            <NItem title="Reddit r/nomie">
              <span slot="right">
                <a
                  href="https://reddit.com/r/nomie"
                  class="btn btn-clear text-primary"
                  target="_system">
                  r/nomie
                </a>
              </span>
            </NItem>

            <NItem title="Open Source">
              <span slot="right">
                <a
                  href="https://github.com/open-nomie/nomie"
                  class="btn btn-clear text-primary"
                  target="_system">
                  Github
                </a>
              </span>
            </NItem>

            <NItem className="compact item-divider" />

            <NItem title="Version">
              <span slot="right" class="pr-2 text-sm">APP_VERSION</span>
            </NItem>
            <NItem title="Built">
              <span slot="right" class="pr-2 text-sm">APP_BUILD_DATE</span>
            </NItem>

          </div>

          <NItem className="compact item-divider" />

          <NItem title={Lang.t('general.questions')}>
            <span slot="right">
              <a
                class="btn btn-clear text-primary"
                href={`mailto:${config.support_email}?subject=Open Nomie Support`}>
                {config.support_contact}
              </a>
            </span>
          </NItem>

          <NItem className="compact item-divider" />

          <NItem title="Reset Nomie">
            <button
              class="btn btn-sm btn-clear text-red"
              slot="right"
              on:click={() => {
                methods.deleteEverything();
              }}>
              Reset
            </button>
          </NItem>

          <NItem className="compact item-divider" />

          <NItem>
            <div class="text-sm pb-2 pt-2">
              &copy; Copyright 2019. All Rights Reserved
            </div>
            <div class="text-sm pb-2">
              Nomie&reg; by
              <a href="https://www.happydata.org" traget="_system">
                Happy Data, LLC
              </a>
            </div>
            <div class="text-sm pb-2" on:click={UserStore.resetLaunchCount}>
              Launch Count {$UserStore.launchCount}
            </div>
          </NItem>

        </div>
        <!-- end container -->

      </div>
    {/if}
  </div>
  <!-- end content slot-->

</AppLayout>

{#if showImporter}
  <ImporterModal on:dismiss={() => (showImporter = false)} />
{/if}
