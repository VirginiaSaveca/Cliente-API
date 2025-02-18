<script setup>
import http from '@/services/http';
import Swal from 'sweetalert2/dist/sweetalert2.js';
import { computed, onMounted, ref } from 'vue';
import { RouterLink, useRoute, useRouter } from 'vue-router';

const router = useRouter();
const route = useRoute();
const links = ref({});
const meta = ref({});

const deleting = ref(false);
const employees = ref([]);
const usp = computed(() => {
  const _ = new URLSearchParams();

  if (typeof route.query?.page) _.append('page', route.query.page);
  if (typeof route.query?.per_page) _.append('per_page', route.query.per_page);

  return _;
});

const processing = ref(false);

const Toast = Swal.mixin({
  toast: true,
  position: 'top-end',
  showConfirmButton: false,
  timer: 3000,
  timerProgressBar: true,
  didOpen: toast => {
    toast.onmouseenter = Swal.stopTimer;
    toast.onmouseleave = Swal.resumeTimer;
  }
});

const GET = async () => {
  processing.value = true;

  try {
    const { data } = await http.get(`/employees${usp.value ? `?${usp.value}` : ''}`);
    employees.value = data.data;
    links.value = data?.links || {};
    meta.value = data?.meta || {};

    console.log(data);
  } catch (error) {
    Toast.fire({
      icon: 'error',
      title: 'Oops...',
      text: 'Algo correu mal!'
    });

    console.log(error);
  } finally {
    processing.value = false;
  }
};

const DELETE = async id => {
  try {
    Swal.fire({
      title: 'Eliminar registo?',
      showDenyButton: false,
      showCancelButton: true,
      confirmButtonText: 'Eliminar'
    }).then(async result => {
      if (result.isConfirmed) {
        deleting.value = true;
        const { data } = await http.delete(`/employees/${id}`);
        Toast.fire({
          icon: 'success',
          title: 'Eliminar',
          text: 'Registo excluido com sucesso.'
        });
        employees.value = employees.value?.filter(employee => employee.id !== id);
        console.log(data);
      }
    });
  } catch (error) {
    if (error?.response) {
      console.log(error.response);

      if (error.response.status === 404) {
        Toast.fire({
          icon: 'error',
          title: 'Oops...',
          // TODO
          text: 'Registo não encontrado.'
        });
      } else if (error.response.status === 429) {
        Toast.fire({
          icon: 'error',
          title: 'Oops...',
          // TODO
          text: 'Muitas requisicoes, aguarde antes de tentar mais uma vez.'
        });
        //certifique-se que estejas online.
      } else {
        Toast.fire({
          icon: 'error',
          title: 'Oops...',
          text: error.response.data?.message
        });
      }
    } else {
      Toast.fire({
        icon: 'error',
        title: 'Oops...',
        text: 'Algo correu mal!'
      });
    }
  } finally {
    deleting.value = false;
  }
};
onMounted(async () => {
  await GET();
});
</script>

<template>
  <div class="row mb-4">
    <div class="col-12">
      <nav aria-label="breadcrumb">
        <ol class="breadcrumb my-0">
          <li class="breadcrumb-item">
            <RouterLink v-bind:to="{ name: 'dashboard' }">Dashboard</RouterLink>
          </li>
          <li class="breadcrumb-item active"><span>Funcionários</span></li>
        </ol>
      </nav>
    </div>
  </div>

  <div class="row mb-2">
    <div class="col-12">
      <RouterLink v-bind:to="{ name: 'employees.create' }" class="btn btn-primary">Criar</RouterLink>
    </div>
  </div>

  <div class="row">
    <div class="col-md-12">
      <div class="card mb-4">
        <div class="card-header">
          <div class="d-flex justify-content-between align-items-center">
            <div class="">
              Funcionários ( {{ employees.length }}/{{ meta?.per_page || 0 }} - {{ meta?.total || 0 }})
            </div>
            <div class="" v-if="processing">Carregando dados ...</div>
          </div>
        </div>

        <div class="card-body">
          <div class="table-responsive">
            <table class="table border mb-0">
              <thead class="fw-semibold text-nowrap">
                <tr class="align-middle">
                  <th class="bg-body-secondary">#</th>
                  <th class="bg-body-secondary">NUIT</th>
                  <th class="bg-body-secondary">Extensão</th>
                  <th class="bg-body-secondary">Nome</th>
                  <th class="bg-body-secondary">Carreira</th>
                  <th class="bg-body-secondary"></th>
                </tr>
              </thead>

              <tbody>
                <tr class="align-middle" v-bind:key="employee.id" v-for="employee in employees">
                  <td>{{ employee.id }}</td>
                  <td>{{ employee.nuit }}</td>
                  <td>{{ employee.branch?.name }}</td>
                  <td>{{ employee.name }}</td>
                  <td>{{ employee.career?.name }}</td>
                  <td>
                    <div class="d-flex gap-2">
                      <RouterLink
                        v-bind:to="{ name: 'employees.show', params: { id: employee.id } }"
                        class="btn btn-ghost"
                      >
                        Ver
                      </RouterLink>

                      <RouterLink
                        v-bind:to="{ name: 'employees.edit', params: { id: employee.id } }"
                        class="btn btn-info"
                      >
                        Editar
                      </RouterLink>

                      <button class="btn btn-danger" type="button" v-on:click="DELETE(employee.id)">Eliminar</button>
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
        <div class="card-footer" v-show="meta?.last_page !== 1">
          <div class="btn-group" role="group">
            <a
              v-bind:href="`/employees?page=${1}`"
              v-if="links?.first && meta?.current_page !== 1"
              class="btn btn-outline-primary"
              >first</a
            >

            <a
              v-bind:href="`/employees?page=${(meta?.current_page || 0) - 1}`"
              v-if="links?.prev && meta?.current_page !== meta?.last_page"
              class="btn btn-outline-primary"
              >prev</a
            >

            <a
              v-bind:href="`/employees?page=${(meta?.current_page || 0) + 1}`"
              v-if="links?.next && meta?.current_page !== meta?.last_page"
              class="btn btn-outline-primary"
              >next</a
            >

            <a
              v-bind:href="`/employees?page=${meta?.last_page || 1}`"
              v-if="links?.last && meta?.current_page !== meta?.last_page"
              class="btn btn-outline-primary"
              >last</a
            >
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style></style>
