<script lang="ts">
  import { allCompositions } from "../aql";
  import { each, text } from "svelte/internal";
  import { useNavigate } from "svelte-navigator";
  import { onMount } from "svelte";
  import { fade, fly } from "svelte/transition";
  import LineChart from "./LineChart.svelte";
  import axios from "axios";

  let temp: { rows: { name: string }[]; columns: Record<string, string>[] } =
    null;
  let labCounter = 0;
  let navigateGrp;
  const navigate = useNavigate();
  let ehrId = window.location.pathname.split("/")[2];
  let searchParams = new URLSearchParams(window.location.search);

  let table = new Set([
    "Time",
    "Temperature",
    "Pulse_Presence",
    "Pulse_Rate",
    "SpO2",
  ]);

  const labelsMap = {
    Temperature: 2,
  };

  const handleLabels = (name, row) => {
    let pos = labelsMap[name];
    let labels = [];
    row.forEach((col) => {
      if (col[pos] != null) {
        labels.push(col[0]);
      }
    });
    return labels;
  };

  onMount(async () => {
    console.log(ehrId);
    const resp = await allCompositions(ehrId);
    console.log(resp);
    temp = resp;

    try {
      await axios.get(
        `http://localhost:8080/ehrbase/rest/openehr/v1/ehr/${ehrId}`,
        {
          headers: {
            Accept: "application/json",
          },
        }
      );
      console.log("EHR exists");
    } catch (e) {
      if (e.response.status === 404) {
        console.log("EHR Does not exist, creating");
        const r = await axios.put(
          `http://localhost:8080/ehrbase/rest/openehr/v1/ehr/${ehrId}`,
          {
            _type: "EHR_STATUS",
            archetype_node_id: "openEHR-EHR-EHR_STATUS.generic.v1",
            name: {
              value: "ehr status",
            },
            subject: {
              external_ref: {
                id: {
                  _type: "GENERIC_ID",
                  value: ehrId,
                  scheme: ehrId,
                },
                namespace: "EHR",
                type: "PERSON",
              },
            },
            is_modifiable: "true",
            is_queryable: "true",
          },
          {
            headers: {
              Accept: "application/json",
            },
          }
        );
        console.log("Created EHR");
      }
    }
  });

  const handleName = (row, key) => {
    if (!row) {
      return "-";
    }

    switch (key) {
      case "Time":
        let time = new Date(row.value);
        return (
          time.getDay().toString() +
          "/" +
          time.getMonth().toString() +
          "/" +
          time.getFullYear().toString() +
          "<br/>" +
          time.getHours().toString() +
          ":" +
          time.getMinutes().toString()
        );

      case "SpO2":
        return row.numerator + "%";

      case "Temperature":
      case "Pulse_Rate":
        return row.magnitude + " " + row.units;
    }
    return row.value;
  };
</script>

<div in:fly={{ y: 200, duration: 500 }}>
  <div
    class="flex flex-row gap-3 p-5 shadow-lg rounded-t-lg border bg-gray-700 justify-between"
  >
    <div>
      <p class="text-2xl text-white">{searchParams.get("Aadhaar")}</p>
      <p class="font-bold text-4xl text-white">{searchParams.get("Name")}</p>
    </div>
    <div class="flex justify-center items-center">
      <sl-button
        type="primary"
        on:click|preventDefault={() => {
          navigate(`/post-data/${ehrId}`);
        }}
        outline
      >
        <sl-icon name="plus-square-fill" slot="prefix" />Add Data
      </sl-button>
    </div>
  </div>

  {#if temp}
    <div class="flex flex-col gap-3 p-5 shadow-lg rounded-b-lg border">
      {#if temp.rows.length != 0}
        <div>
          <p
            class="px-10 py-2 text-white font-bold border rounded text-center text-3xl my-3 {temp
              .rows[0][1].value == 'YES'
              ? 'bg-yellow-500 border-yellow-1000'
              : 'bg-green-500 border-green-1000'}"
          >
            {temp.rows[0][1].value == "YES" ? "Admitted" : "Not Admitted"}
          </p>
        </div>
        <sl-tab-group bind:this={navigateGrp}>
          <sl-tab slot="nav" panel="clinical">Clinical Data</sl-tab>
          <sl-tab slot="nav" panel="vital">Vital Signs</sl-tab>
          <sl-tab slot="nav" panel="travel">Travel History</sl-tab>
          <sl-tab slot="nav" panel="lab">Laboratory Tests</sl-tab>

          <sl-tab-panel name="clinical">
            <h3 class="text-3xl font-bold">Clinical Background</h3>
            <div class="grid grid-cols-2 gap-3 p-5">
              <div
                class="flex flex-col gap-3 p-5 rounded-lg border justify-around"
              >
                <p class="text-center">
                  <span class="font-bold">Symptoms</span>
                  <span
                    class="px-10 py-2 m-5 text-white font-bold border rounded text-center {temp
                      .rows[0][6] != null
                      ? 'bg-red-500'
                      : 'bg-green-500'}"
                    >{temp.rows[0][6] != null
                      ? temp.rows[0][6].value
                      : "N/A"}</span
                  >
                </p>
                <div class="flex flex-row gap-3 p-5 justify-evenly">
                  <p>
                    <span class="font-bold">Screening Purpose</span> :
                    <span
                      >{temp.rows[0][7] != null
                        ? temp.rows[0][7].value
                        : "N/A"}</span
                    >
                  </p>
                  <p>
                    <span class="font-bold">Symptom Detail</span> :
                    <span
                      >{temp.rows[0][8] != null
                        ? temp.rows[0][8].items[0].value.value
                        : "N/A"}</span
                    >
                  </p>
                </div>
              </div>
              <div
                class="flex flex-row gap-3 p-5 rounded-lg border justify-around items-center"
              >
                <p>
                  <span class="font-bold">Present Conditions</span> :
                  <span
                    >{temp.rows[0][9]?.value != null
                      ? temp.rows[0][9].value
                      : "N/A"}</span
                  >
                </p>
                <p>
                  <span class="font-bold">Specific Condition</span> :
                  <span
                    >{temp.rows[0][10] != null
                      ? temp.rows[0][10].items[0].value.value
                      : "N/A"}</span
                  >
                </p>
              </div>
            </div>
          </sl-tab-panel>

          <sl-tab-panel name="vital">
            <div class="flex flex-col gap-3 p-5">
              <h3 class="text-3xl font-bold">Vital Signs</h3>
              <div class="flex flex-col gap-3 p-5 rounded-lg border">
                <table>
                  <tbody>
                    {#each temp.columns as key, i}
                      {#if table.has(key.name)}
                        <tr>
                          <td class="font-bold">
                            {key.name.split("_").join(" ")}
                          </td>
                          {#each temp.rows as row}
                            <td
                              class={key.name == "Time"
                                ? "font-bold text-center pb-5"
                                : "text-center"}
                            >
                              {@html handleName(row[i], key.name)}
                            </td>
                          {/each}
                        </tr>
                      {/if}
                    {/each}
                  </tbody>
                </table>
              </div>
              <div class="flex flex-col border shadow-lg p-5 rounded-lg">
                <p class="text-xl font-bold">Body Temperature Monitoring</p>
                <div class="p-5 m-5">
                  <!-- <LineChart /> -->
                </div>
              </div>
            </div>
          </sl-tab-panel>

          <sl-tab-panel name="travel">
            <div class="flex flex-col gap-3 p-5">
              <h3 class="font-bold text-3xl">Recent Travelling</h3>
              <div class="grid grid-cols-2 gap-3 p-5 rounded-lg border">
                <p class="text-center">
                  <span class="font-bold">Travelled Recently ? </span> :
                  <span
                    class="px-10 py-2 m-5 text-white font-bold border rounded text-center {temp
                      .rows[0][11] != 'Yes'
                      ? 'bg-red-500'
                      : 'bg-green-500'}"
                  >
                    {temp.rows[0][11] != null ? temp.rows[0][11].value : "N/A"}
                  </span>
                </p>

                <p class="text-center">
                  <span class="font-bold">Where ?</span> :
                  <span
                    class="px-10 py-2 m-5 text-white font-bold border rounded text-center {temp
                      .rows[0][12] != null
                      ? 'bg-yellow-500'
                      : 'bg-green-500'}"
                    >{temp.rows[0][12] != null
                      ? temp.rows[0][12].value
                      : "N/A"}</span
                  >
                </p>
              </div>
            </div>
          </sl-tab-panel>

          <sl-tab-panel name="lab">
            <div class="flex flex-col gap-3 p-5">
              <h3 class="font-bold text-4xl mb-5 text-center">
                Laboratory Tests
              </h3>
              {#each temp.rows as test}
                {#if test[13]}
                  <div
                    class="p-5 rounded-lg grid grid-rows-2 shadow-inner bg-gray-800"
                  >
                    <div class="grid grid-cols-2 justify-evenly">
                      <p
                        class="flex flex-col text-center font-bold text-3xl mb-5 text-white"
                      >
                        {test[13].value}
                        <span class="font-normal text-base m-2 text-white"
                          >{@html handleName(test[14], "Time")}</span
                        >
                      </p>

                      <p class="text-center text-xl">
                        <span
                          class="px-10 py-2 m-5 text-white font-bold border-gray-900 border rounded text-center {test[15]
                            ?.value == 'Positive'
                            ? 'bg-red-500'
                            : test[15]?.value == 'Negative'
                            ? 'bg-green-500'
                            : 'bg-yellow-500'}"
                        >
                          {test[15]?.value}
                        </span>
                      </p>
                    </div>
                    <div class="flex justify-evenly">
                      <p
                        class="appearance-none w-full bg-gray-100 text-gray-700 border border-gray-200 rounded py-3 px-4 text-lg"
                      >
                        {test[16] != null ? test[16].value : "No Comments"}
                      </p>
                    </div>
                  </div>
                  <br />
                {/if}
              {/each}
            </div>
          </sl-tab-panel>
        </sl-tab-group>
      {:else}
        <p
          class="font-bold text-3xl text-center rounded-lg border shadow-lg p-5"
        >
          No Data Posted for this Patient
        </p>
      {/if}
    </div>
  {/if}
</div>
