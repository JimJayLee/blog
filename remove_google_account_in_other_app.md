### Change Google account programmatically
In general, if we want to change or log out Google account or other accounts, we have to go Account settings in Settings app and remove the original one. What if we want it done by just one click in any app. Let's get into this.

#### Requirement
Note that source codes which can be downloaded from Android official repo are based on Android P and Xposed framework need installed on the phone already.

#### Exploring the process of account removal in Settings source codes
Start from a keyword that related to the account removal process since there are a lot of files in Settings project and we can not do it from looking up every file. Picking "Remove account" which is labeled on a button, we are led to  _RemoveAccountPreferenceController.java_ class.  It has a onClick method that is believed to be bind to the remove button.

```java
// RemoveAccountPreferenceController.java
    @Override
    public void onClick(View v) {
        if (mUserHandle != null) {
            final EnforcedAdmin admin = RestrictedLockUtils.checkIfRestrictionEnforced(mContext,
                    UserManager.DISALLOW_MODIFY_ACCOUNTS, mUserHandle.getIdentifier());
            if (admin != null) {
                RestrictedLockUtils.sendShowAdminSupportDetailsIntent(mContext, admin);
                return;
            }
        }

        ConfirmRemoveAccountDialog.show(mParentFragment, mAccount, mUserHandle);
    }
```

In the end, it makes sense that a confirmation dialog would be popup when the remove button is clicked. The account removal login gotta be in _ConfirmRemoveAccountDialog_ which is found a nested class in the controller .

```java
/**
     * Dialog to confirm with user about account removal
     */
    public static class ConfirmRemoveAccountDialog extends InstrumentedDialogFragment implements
            DialogInterface.OnClickListener {
        private static final String KEY_ACCOUNT = "account";
        private static final String REMOVE_ACCOUNT_DIALOG = "confirmRemoveAccount";
        private Account mAccount;
        private UserHandle mUserHandle;

        public static ConfirmRemoveAccountDialog show(
                Fragment parent, Account account, UserHandle userHandle) {
            if (!parent.isAdded()) {
                return null;
            }
            final ConfirmRemoveAccountDialog dialog = new ConfirmRemoveAccountDialog();
            Bundle bundle = new Bundle();
            bundle.putParcelable(KEY_ACCOUNT, account);
            bundle.putParcelable(Intent.EXTRA_USER, userHandle);
            dialog.setArguments(bundle);
            dialog.setTargetFragment(parent, 0);
            dialog.show(parent.getFragmentManager(), REMOVE_ACCOUNT_DIALOG);
            return dialog;
        }

        @Override
        public void onCreate(Bundle savedInstanceState) {
            super.onCreate(savedInstanceState);
            final Bundle arguments = getArguments();
            mAccount = arguments.getParcelable(KEY_ACCOUNT);
            mUserHandle = arguments.getParcelable(Intent.EXTRA_USER);
        }

        @Override
        public Dialog onCreateDialog(Bundle savedInstanceState) {
            final Context context = getActivity();
            return new AlertDialog.Builder(context)
                    .setTitle(R.string.really_remove_account_title)
                    .setMessage(R.string.really_remove_account_message)
                    .setNegativeButton(android.R.string.cancel, null) 
                    .setPositiveButton(R.string.remove_account_label, this)
                    .create();
        }
        
        // Just respond to the confirm action
        @Override
        public void onClick(DialogInterface dialog, int which) {
            Activity activity = getTargetFragment().getActivity();
            AccountManager.get(activity).removeAccountAsUser(mAccount, activity,
                    new AccountManagerCallback<Bundle>() {
                        @Override
                        public void run(AccountManagerFuture<Bundle> future) {
                            // codes
	            if (future.getResult()
                                        .getBoolean(AccountManager.KEY_BOOLEAN_RESULT)) {
                                    failed = false;
                                }					
                        }
                    }, null, mUserHandle);
        }
    }
```

Keep in mind that the codes showed is in-completed for some codes are not helpful for analysis. 
As known from codes above, _mAccount_, represented an account entity and hold account info, is passed from controller, with _mUserHandler_ which is for authentication. So the key is calling _AccountManager.get(activity).removeAccountAsUser_.
However, that's not enough, we get to find out where account and UserHandler are passed from. Go on with looking up the _RemoveAccountPreferenceController_ reference.


```java
// AccountDetailDashboardFragment.java
    @Override
    public void onCreate(Bundle icicle) {
        super.onCreate(icicle);
        getPreferenceManager().setPreferenceComparisonCallback(null);
        Bundle args = getArguments();
        final Activity activity = getActivity();
        UserHandle userHandle = Utils.getSecureTargetUser(activity.getActivityToken(),
                (UserManager) getSystemService(Context.USER_SERVICE), args,
                activity.getIntent().getExtras());
        if (args != null) {
            if (args.containsKey(KEY_ACCOUNT)) {
                mAccount = args.getParcelable(KEY_ACCOUNT);
            }
            if (args.containsKey(KEY_ACCOUNT_LABEL)) {
                mAccountLabel = args.getString(KEY_ACCOUNT_LABEL);
            }
            if (args.containsKey(KEY_ACCOUNT_TYPE)) {
                mAccountType = args.getString(KEY_ACCOUNT_TYPE);
            }
        }
        mAccountSynController.init(mAccount, userHandle);
        mRemoveAccountController.init(mAccount, userHandle);
    }
```
 The codes would belong to the account detail page. UserHandler is retrived by a Utils. And account is passed from outside.

```java
// Utils.java
/**
     * Returns the target user for a Settings activity.
     * <p>
     * User would be retrieved in this order:
     * <ul>
     * <li> If this activity is launched from other user, return that user id.
     * <li> If this is launched from the Settings app in same user, return the user contained as an
     *      extra in the arguments or intent extras.
     * <li> Otherwise, return UserHandle.myUserId().
     * </ul>
     * <p>
     * Note: This is secure in the sense that it only returns a target user different to the current
     * one if the app launching this activity is the Settings app itself, running in the same user
     * or in one that is in the same profile group, or if the user id is provided by the system.
     */
    public static UserHandle getSecureTargetUser(IBinder activityToken,
            UserManager um, @Nullable Bundle arguments, @Nullable Bundle intentExtras) {
        UserHandle currentUser = new UserHandle(UserHandle.myUserId());
        IActivityManager am = ActivityManager.getService();
        try {
            String launchedFromPackage = am.getLaunchedFromPackage(activityToken);
            boolean launchedFromSettingsApp = SETTINGS_PACKAGE_NAME.equals(launchedFromPackage);

            UserHandle launchedFromUser = new UserHandle(UserHandle.getUserId(
                    am.getLaunchedFromUid(activityToken)));
            if (launchedFromUser != null && !launchedFromUser.equals(currentUser)) {
                // Check it's secure
                if (isProfileOf(um, launchedFromUser)) {
                    return launchedFromUser;
                }
            }
			
            UserHandle extrasUser = getUserHandleFromBundle(intentExtras);
            if (extrasUser != null && !extrasUser.equals(currentUser)) {
                // Check it's secure
                if (launchedFromSettingsApp && isProfileOf(um, extrasUser)) {
                    return extrasUser;
                }
            }
            UserHandle argumentsUser = getUserHandleFromBundle(arguments);
            if (argumentsUser != null && !argumentsUser.equals(currentUser)) {
                // Check it's secure
                if (launchedFromSettingsApp && isProfileOf(um, argumentsUser)) {
                    return argumentsUser;
                }
            }
        } catch (RemoteException e) {
            // Should not happen
            Log.v(TAG, "Could not talk to activity manager.", e);
        }
        return currentUser;
    }
```

Seems there are some problems in getting a UserHanlder, because removing accounts needs to be a system app.

#### Get Accounts
Firstly, ask for the _GET_ACCOUNTS_ permission. 

```java
// Be sure that permission is granted or a empty list would be returned
final Account[] accounts =  AccountManager.get(context).getAccounts();
Account googleAccount;
for(Account account: accounts){
    if(account.type.equals("com.google"){
         googleAccount = account;
         break;
    }   
}
```
Secondly, get the UserHandler. I tried using _SYSTEM_ UserHandler with reflection.

```java
try{
    Field uhField  = UserHanlder.class.getDeclaredField("SYSTEM");
    uhField.setAccessible(true);
    UserHandler uh = uhField.get(UserHandler.class);
} catch(Execption e){
}
```

That would throw an SecurityException which means the app is not allowed to remove Google account:

```
SecurityException: uid 10156 cannot remove account of type "com.google"
```

#### Hook the Setting app
There might be other ways for accomplishing removal. I make Settings remove the account instead with Xposed. I would not cover the hooking part since it has been mentioned in the previous article.

Therefore, all we need to do is send a remove command to Settings. How to do?

```java
public static void removeAccount(Context context, Account account){
    AccountManager.get(context).removeAccount(account, new AccountManagerCallback<Boolean>(){//...}, null);
}
```

_AccountManage_ provides a more convenient method _removeAccount_ for removing account, more importantly, do not have to care about "Activity" and "UserHandler".  As a result, we don't hook the _removeAccountAsUser_.

Note that _Account_ is parcelable, so it could be sent from other processes. Or getting account info in Settings is totally okay.



