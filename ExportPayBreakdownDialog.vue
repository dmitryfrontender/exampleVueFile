<template>
  <Dialog v-model="open">
    <template #header>
      <div
        class="flex items-center justify-between p-4 w-full bg-white border-b-1 border-gray-200"
      >
        <div class="flex items-center justify-between gap-4">
          <BubbleIcon
            :icon="Icon.FileDownload"
            size="xlarge"
            foreground-class="white"
            background-class="purple-900"
          />
          <div>
            <p class="mb-0 text-gray-600 text-12 text-uppercase">
              {{ $tc('heading.exportPayBreakdownFor') }}
            </p>
            <h3 class="mb-0 leading-5 text-18 font-weight-bold">{{ title }}</h3>
          </div>
        </div>
        <CloseButton
          v-if="viewport.md"
          @click="open = false"
        />
      </div>
    </template>
    <div>
      <v-progress-linear
        v-show="employeePayPeriodSummaries.isLoading"
        indeterminate
        color="#ec3f8c"
      />
      <div class="all:p-4">
        <TimesheetExportFilters
          v-model="form"
          :additional-employee-filter="additionalEmployeeFilter"
        />
        <InlineDialog
          v-if="
            !filteredPayBreakdownSummaries.length &&
            employeePayPeriodSummaries.isLoaded
          "
          class="mt-4"
          :type="InlineDialogTheme.Error"
          :message="$tc('noData.noPayBreakdownsExport')"
        />
        <InlineDialog
          v-else-if="employeePayPeriodSummaries.isLoaded"
          class="mt-4"
          :type="InlineDialogTheme.Info"
          :message="$tc('tooltip.onlyExportApprovedEntries')"
        />
      </div>
    </div>
    <template #footer>
      <div class="flex justify-end gap-2">
        <Button
          :disabled="exporting"
          color="transparent"
          @click="emit('input', false)"
        >
          {{ $t('button.cancel') }}
        </Button>
        <Button
          color="pink"
          :loading="exporting"
          :disabled="
            v$.$invalid ||
            employeePayPeriodSummaries.isLoading ||
            !filteredPayBreakdownSummaries.length ||
            exporting
          "
          @click="exportPayBreakdown()"
        >
          {{ $tc('button.export') }}
        </Button>
      </div>
    </template>
  </Dialog>
</template>

<script setup lang="ts">
import Button from '@/components/buttons/Button.vue';
import CloseButton from '@/components/buttons/dedicated/CloseButton.vue';
import Dialog from '@/components/dialog/Dialog.vue';
import InlineDialog from '@/components/info/InlineDialog.vue';
import BubbleIcon from '@/components/interface/BubbleIcon.vue';
import i18n from '@/i18n';
import { withLoading } from '@/lib/composition/loading';
import { PlainDate } from '@/lib/date-time/PlainDate';
import { employeeCurrent } from '@/lib/employee/employeeFunctions';
import { InlineDialogTheme } from '@/lib/enum/Colour';
import { Icon } from '@/lib/enum/Icon';
import {
  createRealtimeQueryLifetime,
  realtimeQuery,
} from '@/lib/realtime/weak/realtimeFunctions';
import { CustomEntity } from '@/lib/realtime/weak/realtimeTypes';
import { Entity } from '@/lib/store/realtimeEntities';
import {
  buildPayBreakdownExportData,
  payBreakdownCSVFormatter,
} from '@/lib/timesheets/payBreakdownFunctions';
import { TimesheetExportFiltersType } from '@/lib/timesheets/timesheetExportTypes';
import { TimesheetV2FilterType } from '@/lib/timesheetV2/timesheetV2Filter';
import store from '@/store';
import { sortBy } from '@/util/arrayFunctions';
import { requestDownload } from '@/util/domFunctions';
import { getEntity } from '@/util/entityFunctions';
import { buildCSV } from '@/util/exportFunctions';
import { viewport } from '@/util/windowFunctions';
import TimesheetExportFilters from '@/views/timesheetsV2/components/export/TimesheetExportFilters.vue';
import useVuelidate from '@vuelidate/core';
import debounce from 'lodash.debounce';
import spacetime from 'spacetime';
import { computed, onMounted, ref, watch } from 'vue';
import {
  Employee,
  EmployeePayPeriodSummary,
  PayPeriod,
} from '/api/v1';

const props = withDefaults(
  defineProps<{
    payPeriod: PayPeriod;
    value: boolean;
    timesheetFilters?: TimesheetV2FilterType | undefined;
  }>(),
  { timesheetFilters: undefined },
);
const emit = defineEmits<{
  (e: 'input', v: boolean): void;
  (e: 'close'): void;
}>();

const form = ref<TimesheetExportFiltersType>({
  startedAt: props.payPeriod.startsOn.toString(),
  endedAt: props.payPeriod.endsOn.toString(),
  employeeIds: [0],
  jobRoleIds: [props.timesheetFilters?.jobRoleId || 0],
  locationIds: [props.timesheetFilters?.locationId || 0],
  groupIds: [props.timesheetFilters?.employeeGroupId || 0],
  employmentTypes: [props.timesheetFilters?.employmentType || 0],
  tagIds: [props.timesheetFilters?.tagId || 0],
});

const lifetime = createRealtimeQueryLifetime();

const employeePayPeriodSummaries = ref(
  realtimeQuery(CustomEntity.EmployeePayPeriodSummary, {
    startDate: PlainDate.from(form.value.startedAt),
    endDate: PlainDate.from(form.value.endedAt),
  }).empty(),
);

const employees = realtimeQuery(Entity.Employee).fetch();

// use vuelidate state from child LocationForm.vue component
// https://vuelidate-next.netlify.app/advanced_usage.html#nested-validations
const v$ = useVuelidate();

const open = computed<boolean>({
  get: () => {
    return props.value;
  },
  set: (v) => {
    if (!v) emit('close');
    emit('input', v);
  },
});

const timezone = computed<string>(() => store.getters.timezone);

const title = computed<string>(() => {
  return `${i18n.d(
    PlainDate.from(form.value.startedAt).toDate(timezone.value),
    'dayMonth',
  )} - ${i18n.d(
    PlainDate.from(form.value.endedAt).toDate(timezone.value),
    'dayMonthYear',
  )}`;
});

const additionalEmployeeFilter = (e: Employee) => {
  return (
    employeeCurrent(e) ||
    employeePayPeriodSummaries.value.data.some(
      (summary) => summary.employeeId === e.id,
    )
  );
};

const filteredPayBreakdownSummaries = computed<EmployeePayPeriodSummary[]>(
  () => {
    return employeePayPeriodSummaries.value.data.filter((summary) => {
      if (
        !summary.hourlyRatesByJobRole.length ||
        !form.value.employeeIds.some(
          (id) =>
            id === 0 || form.value.employeeIds.includes(summary.employeeId),
        )
      ) {
        return false;
      }
      const employee = getEntity(summary.employeeId, employees.data);
      return (
        employee &&
        form.value.groupIds.some(
          (id) => id === 0 || id === employee.employeeGroupId,
        ) &&
        form.value.employmentTypes.some(
          (type) => type === 0 || type === employee.employmentType,
        )
      );
    }) as EmployeePayPeriodSummary[];
  },
);

const exporting = ref<boolean>(false);
const exportPayBreakdown = withLoading(exporting, async () => {
  const employeeIdsInUse = filteredPayBreakdownSummaries.value.map(
    (summary) => summary.employeeId,
  );
  const employeePayDetailsQuery = realtimeQuery(Entity.EmployeePay)
    .whereIn('id', employeeIdsInUse)
    .fetch({ forceRefetch: true });
  await employeePayDetailsQuery.promise;

  const jobRoleIdsInUse = filteredPayBreakdownSummaries.value.reduce(
    (jobRoleIds, summary) => [
      ...jobRoleIds,
      ...summary.hourlyRatesByJobRole.map((rate) => rate.jobRoleId),
    ],
    [],
  );
  const jobRoleQuery = realtimeQuery(Entity.JobRole)
    .whereIn('id', jobRoleIdsInUse)
    .fetch({ forceRefetch: true });
  await jobRoleQuery.promise;

  const payBreakdownExportData = buildPayBreakdownExportData(
    filteredPayBreakdownSummaries.value as EmployeePayPeriodSummary[],
    employees.data,
    employeePayDetailsQuery.data,
    jobRoleQuery.data,
  );

  requestDownload(
    buildCSV(
      sortBy(payBreakdownExportData, 'name'),
      payBreakdownCSVFormatter(),
    ),
    `pay_breakdown_${spacetime(form.value.startedAt, timezone.value).format(
      'iso-short',
    )}_${spacetime(form.value.endedAt, timezone.value).format(
      'iso-short',
    )}.csv`,
  );
});

const fetchEmployeePayPeriods = (): void => {
  employeePayPeriodSummaries.value = realtimeQuery(
    CustomEntity.EmployeePayPeriodSummary,
    {
      startDate: PlainDate.from(form.value.startedAt),
      endDate: PlainDate.from(form.value.endedAt),
      ...(!form.value.jobRoleIds.includes(0) && {
        jobRoleIds: form.value.jobRoleIds,
      }),
      ...(!form.value.locationIds.includes(0) && {
        locationIds: form.value.locationIds,
      }),
      ...(!form.value.tagIds.includes(0) && {
        tagIds: form.value.tagIds,
      }),
    },
  ).fetch({ lifetime });
};

const fetchEmployeePayPeriodsDebounced = debounce(
  function fetchEmployeePayPeriodsInner() {
    fetchEmployeePayPeriods();
  },
  300,
);

watch(
  form,
  () => {
    if (props.value) {
      fetchEmployeePayPeriodsDebounced();
    }
  },
  { immediate: true, deep: true },
);

onMounted(() => {
  fetchEmployeePayPeriodsDebounced();
});
</script>
