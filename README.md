# angular-reactive-related-field-validator
Angular - Reactive Forms - Have a validator that runs a single assertFn and triggers validation on other fields.

# background
I have a big problem with how control/form level validation is performed in angular Reactive Forms, and have came across this issue multiple times

# the problem

i have a form on a component and, i'd like to ensure that the "startDate" is always *before* the "endDate".
```
form = new FormGroup({
  startDate: new FormControl(),
  endDate: new FormControl()
}) // or however you define it...
```

## scenario a - control level validators (psuedo code)

logically, one would think - hey, lets just create a validator for each control!

```
function startValidateFn(control: AbstractControl): ValidationErrors | null {
  const start = control.value,
    end = control.parent.get('endDate').value;
  return start && end && start > end ? {invalidRange: true} : null;
}

function endValidationFn(control: AbstractControl): ValidationErrors | null {
  const end = control.value,
    start = control.parent.get('startDate').value;
  return start && end && start > end ? {invalidRange: true} : null;
}
```
....
```
form = new FormGroup({
  startDate: new FormControl(null, [startValidateFn]),
  endDate: new FormControl(null, [endValidateFn])
})
```

## scenario a - results

while the FORM will succesfully validate, and the control that is being updated validates, the validationState of the "other" control isn't updated visually.

eg:
```
start = jan 1; // valid
end = feb 1; // valid
// ...
start.setValue(mar 1); // start = invalid, end = valid
// ...
end.setValue(apr 1); // start = valid (but still shows as invalid), end = valid
```

if we force the other control to `.updateValueAndValidity` from the validation routine, we are just adding more and more code to each validation function.  This is fine if we have 1 or 2 simple cases, but gets grosser each time we add another control to the pool.

## scenario a - summary

in short, we have to hand craft each individual validation function, and either let them incorrectly render, or implement even more code to get them to trigger validation on one another (or worse yet, add a buncha garbage to your template to handle and micromanage state manually)

# scenario b - form level validation

so, lets try form level validation:

```
function formValidatorFn(fg: FormGroup): ValidationErrors | null {
  const raw = fg.getRawValue(),
    start = raw.startDate,
    end = raw.endDate;
  return start && end && start > end ? {invalidRange: true} : null;
}
```
...
```
form = new FormGroup({
  startDate: new FormControl(),
  endDate: new FormControl()
}, {
  validators: [formValidatorFn]
})
```

## scenario b - results/summary

similar problem: this runs *after* the control level validators, so by the time this comes, neither control is validated correctly.  so, we need to revalidate!  well, we have the same problem; neither control has its own validation state updated, so we end up having to add validators to the controls also - 3 validators.. dumb (granted, those will no longer need sibling checks in them, but still, 1 + (c^n-1) validators...

# a solution

so, i'm thinking, "why not just create a validator that can take care of this, then pass the info needed at runtime"

## the validator factory

`relatedControlValidator.ts`

```
import { AbstractControl, ValidationErrors, ValidatorFn } from "@angular/forms";

/**
 * @param isValid: function that accepts the "raw" form value and returns a ValidationError | null
 * @param ... controlNames: string[] the listing of all control keys that are dependant on this same assert logic and are connected to one another
 */
export function relatedControlValidator<T>(isValid: (formValue: T) => ValidationErrors | null, ... controlNames: string[]): ValidatorFn {
	const controls: AbstractControl[] = [];
	return function(control: AbstractControl): ValidationErrors | null {

		const form = control.parent;
		if (!form) return null; // gtfo quickly

		if (!controls.length && controlNames.length) {
			controlNames.forEach(name => controls.push(form?.get(name)));
		}
		const validationResponse = isValid(form?.getRawValue());
		if (!validationResponse) { // NULL = valid
			controls
				.filter(c => c !== control && c.invalid)
				.forEach(c => c.updateValueAndValidity({emitEvent: false, onlySelf: true}))
		}
		return validationResponse;
	}
}
```

## the implementation (component)

call our factory function:
```
function dateRowValidator(row: any /* any for testing only */): ValidationErrors | null {
	const from = row.startDate,
		to = row.endDate,
		ret = from && to && from > to ? {invalidRange: true} : null;
	return ret;
}

const dateValidator = relatedControlValidator(dateRowValidator<any>, 'startDate', 'endDate');
```

and bind it to our *controls*

```
form = new FormGroup({
  startDate: new FormControl(null, [myRelatedValidator]),
  endDate: new FormControl(null, [myRelatedValidator])
}) // note - no form level validator needed
```

Now, regardless of how you have the values set, validation is rechecked on any "attached" control (`... controlNames[]`), and the UI updates accordingly

