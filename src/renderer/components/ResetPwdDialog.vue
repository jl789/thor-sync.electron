<template>
    <DialogEx persistent v-model="show" @action:ok="onOk" @action:cancel="close" max-width="420px">
        <v-card ref="card">
            <v-card-title class="subheading">Reset Password</v-card-title>
            <v-card-text>
                <v-form ref="form">
                    <v-text-field
                        v-focus
                        validate-on-blur
                        :rules="[passwordRule]"
                        type="password"
                        :disabled="checking"
                        label="New Password"
                        maxlength="20"
                        v-model="newPassword"
                    ></v-text-field>
                    <v-text-field
                        validate-on-blur
                        :disabled="checking"
                        :rules="[repeatedPasswordRule]"
                        label="Repeat Password"
                        type="password"
                        maxlength="20"
                        v-model="repeatedPassword"
                    ></v-text-field>
                </v-form>
            </v-card-text>
            <v-divider/>
            <v-card-actions>
                <v-btn small :disabled="checking" flat @click="close">Abort</v-btn>
                <v-spacer></v-spacer>
                <v-btn
                    small
                    ref="submit"
                    :flat="!checking"
                    :disabled="checking"
                    @click="resetPwd"
                    class="primary"
                >Save</v-btn>
            </v-card-actions>
        </v-card>
    </DialogEx>
</template>

<script lang="ts">
type Arg = {
    privateKey: Buffer | null,
    id?: number
}
import { Vue, Component, Watch, Prop, Mixins } from 'vue-property-decorator'
import { cry } from 'thor-devkit'
import DialogHelper from '@/renderer/mixins/dialog-helper'
import Account from '@/renderer/mixins/Account'

@Component
export default class ResetPwdDialog extends Mixins(
    Account,
    class extends DialogHelper<Arg, void> { }
) {
    show = false
    checking = false
    password = ''
    newPassword = ''
    repeatedPassword = ''
    step = 1

    @Watch('show')
    showChanged(val: boolean) {
        if (!val) {
            this.$resolve(undefined)
        }
    }

    mounted() {
        this.show = true
    }

    close() {
        if (this.checking) {
            return
        }
        this.show = false
    }

    requirePwd() {
        return this.newPassword.trim() || 'Requires non-empty password'
    }

    passwordRule() {
        return (
            (this.newPassword && this.newPassword.length >= 6) ||
            'Requires at least 6 characters'
        )
    }
    repeatedPasswordRule() {
        return this.repeatedPassword === this.newPassword || 'Password mismatch'
    }

    get valid() {
        return (
            this.passwordRule() === true && this.repeatedPasswordRule() === true
        )
    }

    onOk() {
        const btn = (this.$refs.submit as Vue).$el
        btn.focus()
        btn.click()
    }

    async resetPwd() {
        if (this.checking) {
            return
        }
        if (!(this.$refs.form as any).validate()) {
            return
        }
        this.checking = true

        try {
            if (this.arg.privateKey) {
                const ks = await cry.Keystore.encrypt(
                    this.arg.privateKey,
                    this.newPassword
                )
                BDB.wallets
                    .where('id')
                    .equals(this.arg.id!)
                    .modify({ keystore: ks })

                this.show = false
            }
        } catch (error) {
            LOG.error(error)
        } finally {
            this.checking = false
        }

    }
}
</script>
