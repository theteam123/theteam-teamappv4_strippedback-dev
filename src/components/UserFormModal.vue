<template>
  <div class="fixed inset-0 bg-gray-500 bg-opacity-75 flex items-center justify-center">
    <div class="bg-white rounded-lg p-6 max-w-md w-full">
      <h2 class="text-xl font-bold mb-4">{{ isEditing ? 'Edit User' : 'Create New User' }}</h2>
      <form @submit.prevent="handleSubmit" class="space-y-4">
        <div>
          <label for="user-full-name" class="block text-sm font-medium text-gray-700">Full Name</label>
          <input
            id="user-full-name"
            type="text"
            v-model="formData.full_name"
            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-green-500 focus:ring-green-500"
            required
          />
        </div>
        <div>
          <label for="user-email" class="block text-sm font-medium text-gray-700">Email</label>
          <input
            id="user-email"
            type="email"
            v-model="formData.email"
            :disabled="isEditing"
            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-green-500 focus:ring-green-500"
            required
          />
        </div>
        <div>
          <label for="user-role" class="block text-sm font-medium text-gray-700">Role</label>
          <select
            id="user-role"
            v-model="formData.role"
            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-green-500 focus:ring-green-500"
            required
          >
            <option value="">Select a role</option>
            <option v-for="role in roles" :key="role.id" :value="role.name">
              {{ role.name }}
            </option>
          </select>
        </div>
        <div v-if="isEditing">
          <label for="user-status" class="block text-sm font-medium text-gray-700">Status</label>
          <select
            id="user-status"
            v-model="formData.status"
            class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-green-500 focus:ring-green-500"
          >
            <option value="active">Active</option>
            <option value="inactive">Inactive</option>
            <option value="pending">Pending</option>
            <option value="invited">Invited</option>
          </select>
        </div>
        <div class="flex justify-end gap-3 mt-6">
          <button
            type="button"
            @click="$emit('close')"
            class="px-4 py-2 text-sm font-medium text-gray-700 bg-gray-100 rounded-md hover:bg-gray-200"
          >
            Cancel
          </button>
          <button
            type="submit"
            class="px-4 py-2 text-sm font-medium text-white bg-green-600 rounded-md hover:bg-green-700"
            :disabled="loading"
          >
            {{ isEditing ? 'Update' : 'Create' }}
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue';
import { useAuthStore } from '../stores/auth';
import { supabase } from '../lib/supabase';
import type { Role, CompanyUser } from '../lib/types';

const props = defineProps<{
  isEditing: boolean;
  userData: CompanyUser | null;
}>();

const emit = defineEmits<{
  (e: 'close'): void;
  (e: 'submit'): void;
}>();

const authStore = useAuthStore();
const loading = ref(false);
const roles = ref<Role[]>([]);
const formData = ref({
  full_name: props.userData?.full_name || '',
  email: props.userData?.email || '',
  role: props.userData?.role || '',
  status: props.userData?.status || 'active'
});

// Watch for userData changes to update formData
watch(() => props.userData, (newUserData) => {
  if (newUserData) {
    formData.value = {
      full_name: newUserData.full_name || '',
      email: newUserData.email || '',
      role: newUserData.role || '',
      status: newUserData.status || 'active'
    };
  }
}, { immediate: true });

const fetchRoles = async () => {
  try {
    const { data, error } = await supabase
      .from('roles')
      .select('id, name')
      .or(`company_id.eq.${authStore.currentCompanyId},is_system_role.eq.true`)
      .order('name');

    if (error) throw error;
    roles.value = data || [];
  } catch (err) {
    console.error('Error fetching roles:', err);
    roles.value = [];
  }
};

const handleSubmit = async () => {
  loading.value = true;
  try {
    if (props.isEditing && props.userData) {
      // First check if profile exists
      const { data: existingProfile, error: profileError } = await supabase
        .from('profiles')
        .select('id')
        .eq('id', props.userData.id)
        .single();

      if (profileError && profileError.code !== 'PGRST116') { // PGRST116 is "not found"
        throw profileError;
      }

      // If profile doesn't exist, create it
      if (!existingProfile) {
        const { error: createError } = await supabase
          .from('profiles')
          .insert({
            id: props.userData.id,
            full_name: formData.value.full_name,
            created_at: new Date().toISOString(),
            updated_at: new Date().toISOString()
          });

        if (createError) throw createError;
      } else {
        // Update existing profile
        const { error: updateError } = await supabase
          .from('profiles')
          .update({ 
            full_name: formData.value.full_name,
            updated_at: new Date().toISOString()
          })
          .eq('id', props.userData.id);

        if (updateError) throw updateError;
      }

      // Update user company status
      const { error: statusError } = await supabase
        .from('user_companies')
        .update({ 
          status: formData.value.status
        })
        .eq('user_id', props.userData.id)
        .eq('company_id', authStore.currentCompanyId);

      if (statusError) throw statusError;

      // Get the selected role
      const selectedRole = roles.value.find(r => r.name === formData.value.role);
      if (!selectedRole) {
        throw new Error('Selected role not found');
      }

      // Update role
      const { error: roleError } = await supabase
        .from('user_roles')
        .upsert({ 
          user_id: props.userData.id,
          role_id: selectedRole.id,
          company_id: selectedRole.is_system_role ? null : authStore.currentCompanyId
        }, {
          onConflict: 'user_id,role_id'
        });

      if (roleError) throw roleError;
    } else {
      // Get the selected role
      const selectedRole = roles.value.find(r => r.name === formData.value.role);
      if (!selectedRole) {
        throw new Error('Selected role not found');
      }

      // Call the Edge Function to create the user
      const response = await fetch(`${import.meta.env.VITE_SUPABASE_URL}/functions/v1/admin-create-user`, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${(await supabase.auth.getSession()).data.session?.access_token}`
        },
        body: JSON.stringify({
          email: formData.value.email,
          password: Math.random().toString(36).slice(-8), // Generate random password
          full_name: formData.value.full_name,
          role_id: selectedRole.id,
          company_id: authStore.currentCompanyId,
          created_by: authStore.user?.id
        })
      });

      if (!response.ok) {
        const error = await response.json();
        throw new Error(error.error || 'Failed to create user');
      }
    }

    emit('submit');
  } catch (err) {
    console.error('Error submitting form:', err);
  } finally {
    loading.value = false;
  }
};

onMounted(fetchRoles);
</script>