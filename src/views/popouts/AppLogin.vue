<template>
	<section>
		<PopOutHead v-on:closed="returnResult" id-selector="1" v-on:identity="selectIdentity" :identity="selectedIdentity" />

		<section class="app-login">

			<section>
				<PopOutApp :app="appData" :suffix="account ? $t('popouts.login.suffix') : ''" />


				<!------------------------------------->
				<!----- APP LOGIN REQUIREMENTS -------->
				<!------------------------------------->
				<section class="requirements">

					<!------------------------------------->
					<!-------- SELECT ACCOUNT ------------->
					<!------------------------------------->
					<section class="requirement account" v-if="!loginAll && validAccounts.length">
						<section class="boxes">
							<section class="box account-selector" @click="selectTokenAndAccount">
								<section>
									<figure class="name">{{account.sendable()}}</figure>
									<figure class="network">{{account.network().name}}</figure>
								</section>
								<figure class="chevron fas fa-caret-square-down"></figure>
							</section>
						</section>
					</section>

					<!------------------------------------->
					<!-------- SELECT AUTHORITY ------------->
					<!------------------------------------->
					<section class="dangerous-authority" v-if="!loginAll && account && account.authority === 'owner'">
						{{$t('popouts.login.dangerousPermission')}}
					</section>
					<section class="authorities" v-if="!loginAll && account && account.authorities(false).length > 1">
						<Select bordered="1"
						        :options="account.authorities(false)"
						        :parser="x => x.authority"
						        :iconparser="x => x.authority === 'owner' ? {class:'icon-attention red'} : ''"
						        :selected="account"
						        v-on:selected="x => account = x" />
					</section>

					<!------------------------------------->
					<!---- LOGGING IN WITH ALL ACCOUNTS --->
					<!------------------------------------->
					<section class="requirement all-accounts" v-else-if="loginAll && validAccounts.length">
						<figure class="icon icon-network"></figure>
						<section class="text">
							<label>{{$t('popouts.login.allAccountsFor')}}</label>
							<section class="network-accounts-list">
								<section class="network-accounts" v-for="(network,i) in requestedNetworks">
									<span class="name">{{network.name}} ({{network.accounts(true).length}} {{$tc('generic.accounts', network.accounts(true).length)}})</span>
									<span v-if="i+1 < requestedNetworks.length">,</span>
								</section>
							</section>
						</section>

						<figure class="icon bubble icon-help"></figure>
						<section class="bubble-explainer">
							{{$t('popouts.login.allAccountsDescription', {app:appData.name})}}
						</section>
					</section>


					<!------------------------------------->
					<!----------- NO DATA REQUIRED -------->
					<!------------------------------------->
					<section class="requirement personal" v-if="onlyIdentityLogin">
						<figure class="icon icon-check"></figure>
						<figure class="text">
							<b>{{$t('popouts.login.noInfoNeededTitle')}}</b>
							<br>
							{{$t('popouts.login.noInfoNeededDescription')}}
						</figure>
					</section>


					<!------------------------------------->
					<!----------- NO ACCOUNTS ------------->
					<!------------------------------------->
					<section class="requirement no-accounts" v-if="requestedNetworks.length && !validAccounts.length">
						<figure class="network-name" v-if="savedNetwork">{{savedNetwork.name}}</figure>
						<figure class="text">
							<b>{{$t('popouts.login.noAccountsTitle')}}</b>
							<br>
							{{$t('popouts.login.noAccountsDescription')}}
						</figure>
					</section>






					<!------------------------------------->
					<!----- IDENTITY REQUIREMENTS --------->
					<!------------------------------------->
					<section class="requirement personal" v-if="allRequirementsMet && identityRequirements.length">
						<figure class="icon icon-user"></figure>
						<figure class="text">
							<label>{{$t('popouts.login.personalInformation')}}</label>
							{{identityRequirements}}
						</figure>

						<figure class="icon bubble icon-help"></figure>
						<section class="bubble-explainer">
							{{$t('popouts.login.requestingPersonalInformation', {app:appData.name})}}
						</section>
					</section>
				</section>
			</section>

			<section class="actions">
				<Button big="1" :text="$t('generic.cancel')" @click.native="returnResult(null)" />
				<Button big="1" style="padding:0 20px;" :disabled="!allRequirementsMet" blue="1" :text="$t('generic.allow')" @click.native="login" />
			</section>

		</section>

	</section>
</template>

<script>
	import { mapActions, mapGetters, mapState } from 'vuex'
	import {IdentityRequiredFields} from "@walletpack/core/models/Identity";
	import Network from "@walletpack/core/models/Network";
	import RequiredFields from "../../components/popouts/RequiredFields";
	import PopupService from "../../services/utility/PopupService";
	import {Popup} from "../../models/popups/Popup";
	import {Blockchains} from "@walletpack/core/models/Blockchains";
	import * as ApiActions from '@walletpack/core/models/api/ApiActions';
	import PopOutApp from "../../components/popouts/PopOutApp";
	require('../../styles/transfers.scss');

	export default {
		props:['popup', 'expanded'],
		components:{
			PopOutApp,
			RequiredFields,
		},
		data () {return {
			account:null,

			selectedAccounts:[],
			searchTerms:'',
			selectedLocation:null,
			selectedIdentity:null,
			showingAll:false,

			loginAll:false,

			reputation:null,
		}},
		created(){
			this.loginAll = this.popup.data.type === ApiActions.LOGIN_ALL;

			if(this.validAccounts.length) this.account = this.validAccounts[0];

			this.selectIdentity(this.identities.sort((a,b) => {
				return b.id === this.scatter.keychain.lastUsedIdentity ? 1 : a.id === this.scatter.keychain.lastUsedIdentity ? -1 : 0;
			})[0]);
		},
		computed: {
			...mapState([
				'scatter',
				'balances',
			]),
			...mapGetters([
				'identity',
				'identities',
				'accounts',
				'networks',
				'locations'
			]),

			appData(){
				return this.popup.data.props.appData;
			},



			validAccounts(){
				if(!this.accountRequirements.length) return [];

				const network = this.accountRequirements.map(x => Network.fromJson(x))[0];
				return network.accounts()
					.sort((a,b) => b.authority === 'active' ? 1 : 0)
					.sort((a,b) => b.logins - a.logins)

			},
			requestedNetworks(){
				return this.accountRequirements.map(raw => {
					const n = Network.fromJson(raw);
					return this.networks.find(x => x.unique() === n.unique());
				});
			},
			network(){
				return Network.fromJson(this.accountRequirements[0] || {})
			},
			savedNetwork(){
				return this.networks.find(x => x.unique() === this.network.unique());
			},
			payload(){ return this.popup.payload(); },
			isValidIdentity() {
				return this.selectedIdentity.hasRequiredFields(this.fields, this.selectedLocation);
			},
			fields() {
				return IdentityRequiredFields.fromJson(this.payload.fields);
			},
			personalFields(){
				return this.fields.personal;
			},
			locationFields(){
				return this.fields.location;
			},
			missingFields(){
				if(!this.personalFields.length && !this.locationFields.length) return false;
				return !this.identity.hasRequiredFields(this.fields);
			},
			identityRequirements() {
				return this.fields.personal.concat(this.fields.location).join(', ');
			},
			accountRequirements() {
				return this.fields.accounts || [];
			},
			allRequirementsMet(){
				return !this.accountRequirements.length || !!this.validAccounts.length;
			},
			onlyIdentityLogin(){
				return !this.fields.personal.length
					&& !this.fields.location.length
					&& !this.fields.accounts.length
			}
		},
		methods: {
			returnResult(result){
				this.$emit('returned', result);
			},



			selectTokenAndAccount(){
				PopupService.push(Popup.selectAccount(account => {
					if(!account) return;
					this.account = account;
				}, this.validAccounts))
			},

			login(){
				this.returnResult({
					identity:this.selectedIdentity,
					location:this.selectedLocation,
					accounts:this.account ? [this.account] : [],
					missingFields:this.missingFields
				});
			},

			selectIdentity(identity){
				this.selectedIdentity = identity.clone();
				if(identity.getLocation()){
					this.selectedLocation = identity.getLocation().clone();
				} else {
					if(this.locations.length) this.selectedLocation = this.locations[0].clone();
				}
			},

		}
	}
</script>

<style scoped lang="scss" rel="stylesheet/scss">
	@import "../../styles/variables";

	.app-login {
		display:flex;
		justify-content: center;
		align-items: center;
		border:1px solid #dadada;
		border-top:0;
		height:calc(100vh - 40px);

		.app-details {
			margin-top:-60px;
		}

		.requirements {
			min-width:400px;
			text-align:left;
			max-width:80%;
			margin:10px auto;

			.authorities {
				.select {
					margin-top:-7px;
				}
			}

			.dangerous-authority {
				background:$red;
				border:1px solid $darkred;
				color:$white;
				font-size: $small;
				margin-bottom:10px;
				margin-top:-7px;
				padding:10px;
				border-radius:$radius;
			}

			.boxes {
				width:100%;

				.box {
					width:100%;
				}

			}

			.requirement {
				padding:10px 0;
				display:flex;
				align-items: center;
				position: relative;

				label {
					font-size: $small;
					padding-top:2px;
				}

				.icon {
					padding-right:5px;
					align-self: flex-start;
					color:$silver;
					margin-left:-8px;

					&.bubble {
						padding:3px 2px;
						border-radius:50%;
						background:$blue;
						border:1px solid $darkerblue;
						color:$white;
						font-size: $tiny;
						cursor: pointer;

						&:hover {
							~ .bubble-explainer {
								display:block;
							}
						}
					}
				}

				.bubble-explainer {
					position:absolute;
					right:-10px;
					bottom:calc(100% - 10px);
					width:380px;
					font-size: $small;
					background:$white;
					color:$black;
					box-shadow:0 2px 4px $blue-shadow, 0 8px 24px $blue-shadow;
					padding:20px;
					border-radius:$radius;
					display:none;
					z-index:999999;
				}

				.text {
					flex:1;
					font-size: $small;
				}

				&.all-accounts {
					margin-top:10px;
					padding-top:20px;
					border-top:1px solid $lightgrey;

					.icon {
						&:first-child {
							color:$blue;
						}
					}

					.network-accounts-list {
						max-height:100px;
						overflow-y:auto;
					}

					.network-accounts {
						font-size: $small;
						font-weight: bold;
						display:inline-block;
						margin-right:5px;

						.name {
							color:$blue;
							text-decoration: underline;
						}
					}
				}

				&.personal {
					margin-top:10px;
					padding-top:20px;
					border-top:1px solid $lightgrey;
				}

				&.no-accounts {
					text-align:center;
					width:350px;
					display:flex;
					justify-content: center;
					align-items: center;
					flex-direction: column;
					border:1px solid $blue;
					border-radius:$radius;
					margin:20px auto 10px;
					padding:20px;

					.network-name {
						font-size: $large;
						font-weight: bold;
						margin-bottom:5px;
					}
				}
			}
		}

		.actions {
			margin-top:30px;
			position:absolute;
			bottom:30px;
			right:30px;
			left:30px;
			display:flex;
			justify-content: space-between;
		}
	}





</style>
