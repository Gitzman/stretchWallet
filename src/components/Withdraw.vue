<template>
<transition name='fade'>
  <div class='withdrawcomp' v-show='$store.state.vaultExist'>
    <div class='inputs'>
      <div class='content'>
        <select v-model='symbol' :disabled='xdrEnvelope != null'>
          <option value='initial' disabled selected>Choose your token</option>
          <option v-for='token in tokens' v-if='token.text!="undefined"' v-bind:value='token.value'>
            {{token.text}}
          </option>
        </select>
      </div>
      <div class='container'></div>
      <div class='content input-field'>
        <input placeholder='Amount' type='number' v-model.number='amount' :disabled='xdrEnvelope != null'>
        </input>
      </div>
    </div>

    <transition name='fade'>
      <div v-if='xdrEnvelope'>
        <form class="col s12">
          <div class="row xdrEnvelope">
            <div class='summary'>
              <label>Summary</label>
              <p>{{summary}}</p>
            </div>
            <div>
              <a class="btn waves-effect light-blue darken-3" target="_blank" :href='laboratoryLink'>
                Sign and withdraw in the stellar laboratory
            </a>
            </div>
          </div>
        </form>
        <div>
          <label>or sign here</label>
        </div>
        <div class='input-field'>
          <input v-model='userPrivateKey' type='text' placeholder='User Private Key'>
          </input>
          <input v-model='vaultPrivateKey' type='text' placeholder='Vault Private Key'>
          </input>
        </div>
      </div>
    </transition>
    <div>
      <transition name='fade'>
        <label class='errorLabel' v-if='!validAmount'>Exceeded Maximum Amount.</label>
      </transition>
    </div>
    <a @click='createTransaction()' v-if='xdrEnvelope == null' :class="{'btn-large waves-effect light-blue darken-3':true, 'disabled': !validAmount}">
      <i class="material-icons right">send</i> Create Transaction
    </a>

    <a @click='submitTransaction()' v-if='xdrEnvelope != null && withdrawStage == "button"' class="btn-large waves-effect light-blue darken-3">
      <i class="material-icons right">send</i> Sign and Withdraw
    </a>
    <div class="preloader-wrapper big active" v-if='withdrawStage === "loader"'>
      <div class="spinner-layer spinner-blue-only">
        <div class="circle-clipper left">
          <div class="circle"></div>
        </div>
        <div class="gap-patch">
          <div class="circle"></div>
        </div>
        <div class="circle-clipper right">
          <div class="circle"></div>
        </div>
      </div>
    </div>
    <div v-if='withdrawStage === "check" || withdrawStage === "close"'>
      <i class='material-icons'>{{withdrawStage}}</i>
    </div>
  </div>
</transition>
</template>



<script>
import jquery from 'jquery';
import StellarSdk from 'stellar-sdk';

const kp = StellarSdk.Keypair;

export default {
  name: 'Withdraw',
  data() {
    return {
      amount: null,
      symbol: null,
      denomination: 1,
      vaultPrivateKey: null,
      userPrivateKey: null,
      xdrEnvelope: null,
      withdrawStage: 'button',
    };
  },
  computed: {
    balances() {
      const res = Object.keys(this.$store.state.balances);
      return res;
    },
    tokens() {
      var res = [];
      const balanceKeys = Object.keys(this.$store.state.balances);
      for (var i = 0; i < balanceKeys.length; i++) {
        res.push({
          'text': balanceKeys[i],
          'value': balanceKeys[i]
        })
      }
      return res
    },
    laboratoryLink: function() {
      if (this.$store.state.networkPassphrase === 'PUBLIC') {
        return `https://www.stellar.org/laboratory/#txsigner?xdr=${encodeURIComponent(this.xdrEnvelope)}&network=public`;
      } else {
        return `https://www.stellar.org/laboratory/#txsigner?xdr=${encodeURIComponent(this.xdrEnvelope)}&network=test`;
      }
    },
    summary: function() {
      return `You are withdrawing ${parseFloat(this.amount)} XLM from the "${this.symbol}" vault. Your final personal balance will be ${parseFloat(this.$store.state.balances["undefined"][0].balance) + this.amount} XLM`
    },
    validAmount: function() {
      //iterate over all the safes from this symbol. at the end of every iteration check if residue is >= 0
      if (this.symbol != null) {
        var counter = this.amount;
        for (var i = 0; i < this.$store.state.balances[this.symbol].safes.length; i++) {
          counter -= this.$store.state.balances[this.symbol].safes[i].amount;
          if (counter <= 0) {
            return true;
          }
        }
        return false;
      }
      return true;

    }
  },
  mounted() {},
  methods: {
    submitTransaction() {
      const server = new StellarSdk.Server(this.$store.state.networkURL);
      if (this.$store.state.networkURL === 'https://horizon.stellar.org') {
        StellarSdk.Network.usePublicNetwork();
      } else {
        StellarSdk.Network.useTestNetwork();
      }
      var transaction = new StellarSdk.Transaction(this.xdrEnvelope);
      var privKP = kp.fromSecret(this.userPrivateKey);
      var vaultprivKP = kp.fromSecret(this.vaultPrivateKey);
      transaction.sign(privKP);
      transaction.sign(vaultprivKP);
      this.withdrawStage = 'loader';
      server.submitTransaction(transaction)
        .then(data => {
          Materialize.toast("SUCCESS: Withdrawal Complete", 2000);
          Materialize.toast(data._links.transaction.href, 2000);
          this.withdrawStage = 'check';
          this.$router.push('/')
        })
        .catch(err => {
          console.log(err);
          Materialize.toast('Failure, Not possible to withdraw', 2000)
          this.withdrawStage = 'close'
          this.$router.push('/')
        })

    },
    createTransaction() {
      const server = new StellarSdk.Server(this.$store.state.networkURL);
      const symbol = this.symbol;
      const safePublicKey = this.$store.state.balances[this.symbol].safes[0].issuer;
      const safeAsset = new StellarSdk.Asset(this.symbol, safePublicKey)
      const storedAsset = new StellarSdk.Asset('XLM', null);
      const balances = this.$store.state.balances;
      const vaultPublicKey = this.$store.state.newVault.publicKey;
      const personalPublicKey = this.$store.state.publicKey;
      const denomination = this.denomination;

      function getDenomination(issuer) {

      }

      function preTransaction(balances, symbol, amount) {
        var arrNeededSafes = [];
        for (var i = 0; i < balances[symbol].safes.length && amount > 0; i++) {
          if (amount > balances[symbol].safes[i].amount) {
            if (balances[symbol].safes[i].amount > 0) {
              arrNeededSafes.push([balances[symbol].safes[i].issuer, balances[symbol].safes[i].amount]);
              amount -= balances[symbol].safes[i].amount
            }
          } else {
            arrNeededSafes.push([balances[symbol].safes[i].issuer, amount])
            amount = 0
          }
        }
        console.log('amounts with safes keys: ', arrNeededSafes);
        return arrNeededSafes;
      }

      function generateOperationsData(balances, symbol, amount, vaultPublicKey) {
        var safesAmount = preTransaction(balances, symbol, amount)
        var arrOperationsData = []
        for (var i = 0; i < safesAmount.length; i++) {
          const safePublicKey = safesAmount[i][0];
          const safeAsset = new StellarSdk.Asset(symbol, safePublicKey)
          const storedAsset = new StellarSdk.Asset('XLM', null);

          const ops1 = {
            'source': vaultPublicKey,
            'selling': safeAsset,
            'buying': storedAsset,
            'amount': (safesAmount[i][1] / denomination) + '',
            'price': 1,
          };
          arrOperationsData.push(ops1);
        }
        console.log('generatedOperationsData:', arrOperationsData);
        return arrOperationsData;
      }


      function generateOperations(balances, symbol, amount, vaultPublicKey) {
        var ops = generateOperationsData(balances, symbol, amount, vaultPublicKey);
        var arrWithdrawOffers = [];

        for (var i = 0; i < ops.length; i++) {
          var withdrawOffer = StellarSdk.Operation.manageOffer(ops[i]);
          arrWithdrawOffers.push(withdrawOffer);
        }
        return arrWithdrawOffers;
      }

      function createEnvelope(balances, symbol, amount, vaultPublicKey, personalKey) {
        return new Promise(function(resolve, reject) {
          var withdrawOperations = generateOperations(balances, symbol, amount, vaultPublicKey);

          const safePublicKey = balances[symbol].safes[0].issuer;
          const safeAsset = new StellarSdk.Asset(symbol, safePublicKey)
          const storedAsset = new StellarSdk.Asset('XLM', null);

          // operation data to return all the lumens from the vault to the personal account
          const ops2 = {
            'source': vaultPublicKey,
            'destination': personalKey,
            'amount': amount + '',
            'asset': new StellarSdk.Asset('XLM', null)
          };

          var payTransactionWallet = StellarSdk.Operation.payment(ops2);

          // start the creation of the transaction and add the withdraw operations
          server.loadAccount(vaultPublicKey)
            .then(accountresp => {

              const sequence = accountresp.sequenceNumber() + '';

              const msg = new StellarSdk.Memo('text', 'Withdrawing from safe');

              const vaultAccount = new StellarSdk.Account(vaultPublicKey, sequence)
              var transaction = new StellarSdk.TransactionBuilder(vaultAccount)

              for (var i = 0; i < withdrawOperations.length; i++) {
                const currentOp = withdrawOperations[i];
                transaction = transaction.addOperation(currentOp);
              }

              transaction = transaction.addOperation(payTransactionWallet)
                .addMemo(msg)
                .build();
              var xdrEnvelope = transaction.toEnvelope().toXDR().toString("base64");
              resolve(xdrEnvelope);
              // return xdrEnvelope;

            })
            .catch(err => {
              console.log('error:', err);
              return 'error'
            });
        });
      }
      createEnvelope(balances, symbol, this.amount, vaultPublicKey, personalPublicKey)
        .then(data => this.xdrEnvelope = data)
    },
  }
};
</script>

<style scoped>
.materialize-textarea {
  overflow-y: scroll;
  margin: 0;
}

select {
  display: block;
}

.inputs {
  display: flex;
}

.container {
  width: 1rem;
}

.content {
  flex: 1;
}

.coltextarea {
  margin: auto;
  width: 77% !important;
}

.xdrEnvelope {
  width: 100%;
}


.withdrawcomp {
  display: block;
  padding: 1rem;
  width: 60%;
  min-width: 597px;
  /* height: 15rem; */
  /* max-height: 50rem; */
  /* min-height: 45rem; */
  margin: auto;
  width: 50%;
  /* height: 50%; */
  background-color: white;
  -webkit-box-shadow: 0 8px 10px 1px rgba(0, 0, 0, 0.14), 0 3px 14px 2px rgba(0, 0, 0, 0.12), 0 5px 5px -3px rgba(0, 0, 0, 0.3);
  box-shadow: 0 8px 10px 1px rgba(0, 0, 0, 0.14), 0 3px 14px 2px rgba(0, 0, 0, 0.12), 0 5px 5px -3px rgba(0, 0, 0, 0.3);
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity .5s;
}

.fade-enter,
.fade-leave-to
/* .fade-leave-active below version 2.1.8 */

  {
  opacity: 0;
}

/* label focus color */

.input-field input[type=text]:focus+label {
  color: #337ab7;
}

/* label underline focus color */

.input-field input[type=text]:focus {
  border-bottom: 1px solid #337ab7;
  box-shadow: 0 1px 0 0 #337ab7;
}

/* label underline focus color */

.input-field input[type=number]:focus {
  border-bottom: 1px solid #337ab7;
  box-shadow: 0 1px 0 0 #337ab7;
}

.input-field {
  margin: 0;
}

.signingkeys {
  margin-top: 0;
}

#amountinput {
  margin-bottom: 1rem;
}

.summary p {
  margin: auto;
  margin-bottom: 1rem;
  width: 70%;
  font-weight: 800;
  font-size: 1.2rem;
}

.summary {
  margin-top: 0rem;
}

.errorLabel {
  color: red;
}
</style>
