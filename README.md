Purpose
--------------

FXForms is an Objective-C library for easily creating table-based forms on iOS. It is ideal for settings pages, or user data entry tasks.

Unlike other solutions, FXForms works directly with strongly-typed data models that you supply (instead of dictionaries or complicated dataSource protocols), and infers as much information as possible from your models using introspection, to avoid the need for tedious duplication of type information.


Supported iOS & SDK Versions
-----------------------------

* Supported build target - iOS 7.0 (Xcode 5.0)
* Earliest supported deployment target - iOS 5.0
* Earliest compatible deployment target - iOS 4.3

NOTE: 'Supported' means that the library has been tested with this version. 'Compatible' means that the library should work on this iOS version (i.e. it doesn't rely on any unavailable SDK features) but is no longer being tested for compatibility and may require tweaking or bug fixes to run correctly.


ARC Compatibility
------------------

FXForms requires ARC. If you wish to use FXForms in a non-ARC project, just add the -fobjc-arc compiler flag to the FXForms.m class. To do this, go to the Build Phases tab in your target settings, open the Compile Sources group, double-click FXForms.m in the list and type -fobjc-arc into the popover.

If you wish to convert your whole project to ARC, comment out the #error line in FXForms.m, then run the Edit > Refactor > Convert to Objective-C ARC... tool in Xcode and make sure all files that you wish to use ARC for (including FXForms.m) are checked.


Creating a form
------------------

To create a form object, just make any new NSObject subclass that conforms to the FXForm protocol, like this:

    @interface MyForm : NSObject <FXForm>
    
    @end

The FXForm protocol has no compulsory methods or properties, so that's literally all you need to do. The FXForms library will inspect your object and identify all public and private properties and use them to generate the form. For example, suppose you wanted to have a form containing an "Email" and "Password" field,  and a "Remember Me" switch; you would define it like this:

    @interface MyForm : NSObject <FXForm>
    
    @property (nonatomic, copy) NSString *email;
    @property (nonatomic, copy) NSString *password;
    @property (nonatomic, assign) BOOL rememberMe;

    @end

That's literally all you have to do. FXForms is *really* smart - much smarter than you would expect. For example:

* Fields will appear in the same order you declare them in your class
* Fields will automatically be assigned suitable control types, for example, the rememberMe field will be displayed as a UISwitch, the email field will automatically have a keyboard of type UIKeyboardTypeEmailAddress and the password field will automatically have secureTextEntry enabled. 
* Field titles are based on the key name, but camelCase is automatically converted to a Title Case, with intelligent handling of ACRONYMS, etc.
* Modifying values in the form will automatically assign those values back to your model object. You can use custom setter methods or KVO to intercept the changes if you nee to perform additional logic.
* If your form contains subforms (properties that conform to the FXForm protocol), they will automatically be instantiated if they are nil - no need to set default values.

These default behaviors are all inferred by inspecting the property type and name using Objective-C's runtime API, but they can also all be overridden if you wish - that's covered later under Tweaking form behavior


Displaying a form (basic)
----------------------------

To display your form in a view controller, you have two options: FXForms provides a UIViewController subclass called FXFormViewController that is designed to make getting started as simple as possible. To set up FXFormViewController, just create it as normal and set your foam as follows:

    FXFormViewController *controller = [[FXFormViewController alloc] init];
    controller.formController.form = [[MyForm alloc] init];

You can then display the form controller just as you would do any ordinary view controller. FXFormViewController contains a UITableView, which it will create automatically as needed. If you prefer however, you can assign your own UITableView to the tableView property and use that instead. You can even initialize the  FXFormViewController with a nib file that creates the tableView.

It is a good idea to place the FXFormViewController inside a UINavigationController. This is not mandatory, but if the form contains subforms, these will be pushed onto its navigationController, and if that does not exist, the forms will not be displayed.

Like UITableViewController, FXFormViewController will normally assign the tableView as the main view of the controller. Unlike UITableViewController, it doesn't *have* to be - you can make your tableView a subview of your main view if you prefer.

Like UITableViewController, FXFormViewController implements the UITableViewDelegate protocol, so if you subclass it, you can override the UITableViewDelegate and UIScrollViewDelegate methods to implement custom behaviors. FXFormViewController is not actually the direct delegate of the tableView however, it is the delegate of its formController, which is an instance of FXFormController. The formController acts as the tableView's delegate and dataSource, and proxies the UITableViewDelegate methods back to the FXFormViewController via the FXFormControllerDelegate protocol.

Unlike UITableViewController, FXFormViewController does not implement UITableViewDataSource protocol. This is handled entirely by the FXFormController, and it is not recommended that you try to override or intercept any of the datasource methods.


Displaying a form (advanced)
----------------------------

The FXFormViewController is pretty flexible, but sometimes it's inconvenient to be forced to use a particular base controller class. For example, you may wish to use a common base class for all your view controllers, or display a form inside a view that does not have an associated controller.

In the former case, you could add an FXFormViewController as a child controller, but in the latter case that wouldn't work. To use FXForms without using FXFormViewController, you can use the FXFormController directly. To display a form using FXFormController, you just need to set the form and tableView properties, and it will do the rest. You can optionally bind the FXFormController's delegate property to be notified of UITableView events.

Here is example code for a custom form view controller:

    @interface MyFormViewController : UIViewController <FXFormControllerDelegate>
    
    @property (nonatomic, strong) IBOutlet UITableView *tableView;
    @property (nonatomic, strong) FXFormController *formController;

    @end
    
    @implementation MyFormViewController
    
    - (void)viewDidLoad
    {
        [super viewDidLoad];
        
        //we'll assume that tableView has already been set via a nib or the -loadView method
        self.formController = [[FXFormController alloc] init];
        self.formController.tableView = self.tableView;
        self.formController.delegate = self;
        self.formController.form = [[MyForm alloc] init];
    }
    
    @end


Tweaking form behavior
------------------------

FXForm's greatest strength is that it eliminates work by guessing as much as possible. It can't guess everything however, and it sometimes guesses wrong. So how do you correct it?

You may find that you don't want all of your object properties to become form fields; you may have private properties that are used internally by your form model for example, or you may just wish to order the fields differently to how you've laid out your properties in the interface.

To override the list of form fields, implement the optional -fields method of your form:

    - (NSArray *)fields
    {
        return @[@"field1", @"field2", @"field3"];
    }

The fields method should return an array of strings, dictionaries, or a mixture. In the example we have returned strings; these map to the names of properties of the form object. If you return an array of names like this, these fields will replace the automatically generated field list.

The -fields method will be called again every time the form is reassigned to the formController. That means that you can generate the fields dynamically, based on application logic. For example, you could show or hide certain fields based on other properties.

In addition to omitting and rearranging fields, you may wish to override their attributes. There are two ways to do this: One way is to add a method to the form object, such as -(NSDictionary *)<name>Field; where name is the property that the field relates to. This method returns a dictionary of properties that you wish to override (see Form field properties, below). For example, if we wanted to override the title of the email field, we could do it like this:

    - (NSDictionary *)emailField
    {
        return @{FXFormFieldTitle: @"Email Address"};
    }

Alternatively, you can return a dictionary in the -fields array instead of a string. If you do this, you must include the FXFormFieldKey so that FXForms knows which field you are overriding:

    - (NSArray *)fields
    {
        return @[
                 @{FXFormFieldKey: @"email", FXFormFieldTitle: @"Email Address"},
                 …other fields…
                ];
    }

These two approaches are equivalent.

Finally, you may wish to add additional, virtual form fields (e.g. buttons or labels) that don't correspond to any properties on your form class. You can do this by implementing the -fields method, but if you're happy with the default fields and just want to add some extra fields at the end, you can override the -extraFields method instead, which works the same way, but leaves in place the default fields inferred from the form class:

    - (NSArray *)extraFields
    {
        return @[
                 @{FXFormFieldTitle: @"Extra Field"},
                ];
    }


Grouping fields
---------------------

You may wish to group your form fields into sections in the form to make it ease to use. FXForms handles grouping in a very simple way - you just add an FXFormFieldHeader or FXFormFieldFooter attribute to any field and it will start/end the section at that point. The FXFormFieldHeader/Footer is a string that will be displayed as the header or footer text for the section. If you don't want any text, just supply an empty string.

In the following example, we have four fields, and we've split them into two groups, each with a header:

    - (NSArray *)fields
    {
        return @[
                 @{FXFormFieldKey: @"field1", FXFormFieldHeader: @"Section 1"},
                 @"field2",
                 @{FXFormFieldKey: @"field3", FXFormFieldHeader: @"Section 2"},
                 @"field4",
                ];
    }

Alternatively, since we aren't overriding any other field properties, we could have done this more cleanly by using the following approach:

    - (NSDictionary *)field1Field
    {
        return @{FXFormFieldHeader: @"Section 1"};
    }

    - (NSDictionary *)field3Field
    {
        return @{FXFormFieldHeader: @"Section 2"};
    }


Form field properties
------------------------

The list of form field properties that you can set are as follows. Most of these have sensible defaults set automatically. Note that the string values of these constants are declared in the interface - you can assume that the string values of these constants will no change in future releases, and you can safely use these values in (for example) a plist used to configure the form.

    static NSString *const FXFormFieldKey = @"key";
    
This is the name of the related property of the form object. If your field isn't backed by a real property, this might be the name of a getter method used to populate the field value. It's also possible to have completely virtual fields (such as buttons) that do not have a key at all.
    
    static NSString *const FXFormFieldType = @"type";
    
This is the field type, which is used to decide how the field will be displayed in the table. The type is used to determine which type of cell to use to represent the field, but it may also be used to configure the cell. The type is automatically inferred from the field property declaration, but can be overridden. Supported types are listed under Form field types below, however you can supply any string as the type and implement a custom form cell to display and/or edit it.
    
    static NSString *const FXFormFieldCell = @"cell";
    
This is the class name of the cell used to represent the field. By default this value is not specified on the field-level; instead, the FXFormController maintains a map of fields types to cells, which allows you to override the default cells used to reprint a given field type on a per-form level rather than having to do it per-field. If you *do* need to provide a special one-off cell type, you can use this property to do so. The value provided can be either a Class object or a string representing the class name.
    
    static NSString *const FXFormFieldTitle = @"title";
    
This is the display title for the field. This is automatically generated from the key by converting from camelCase to Title Case, and then localised by running it through the NSLocalizedString() macro. That means that instead of overriding the title using this key, you can do so in your strings file instead if you prefer.
    
    static NSString *const FXFormFieldAction = @"action";
    
This is an optional action to be performed when by the field. The value represents the name of a selector that will be called when the field is activated. The target is determined by cascading up the responder chain from the cell upwards until an object is encountered that responds to the selector. That means that you could choose to implement this action method on the cell, or on the tableview, or it's superview, or the view controller, or the app delegate, or even then window.

For non-interactive fields, the action will be called when the cell is selected; for fields such as switches or textfields, it will fire when the value is changed. The action method can accept either zero or one argument. The argument supplied will be the form field cell, (a UITableViewCell conforming to the FXFormFieldCell protocol), from which you can access the FXFormField model.
    
    static NSString *const FXFormFieldOptions = @"options";
    
For any field type, you can supply an array of supported values, which will override the standard field with a checklist of options to be selected instead. The options can be NSStrings, NSNumbers or any other object type. If you use a custom object for the values, you can provide a -(NSString *)fieldDescription; method to control how it is displayed in the list. See Form field options below for more details.
    
    static NSString *const FXFormFieldHeader = @"header";
    
This property provides an optional section header string to display before the field. Supply an empty string to create a section partition without a title.
    
    static NSString *const FXFormFieldFooter = @"footer";
    
This property provides an optional section footer string to display after the field. Supply an empty string to create a section partition without a footer.
    
    static NSString *const FXFormFieldInline = @"inline";

Fields whose values is another FXForm, or which have a supplied options array, will normally be displayed in a new form, which is pushed onto the navigation stack when you tap the field. You may wish to display such fields inline within same tableView instead. You can do this by setting the FXFormFieldInline property to @YES.


Form field types
------------------------

    static NSString *const FXFormFieldTypeDefault = @"default";
    
This is the default field type, used if no specific type can be determined.
    
    static NSString *const FXFormFieldTypeLabel = @"label";
    
This type can be used if you want the field to be treated as non-interactive/read-only.
    
    static NSString *const FXFormFieldTypeText = @"text";
    
By default, this field type will be represented by an ordinary UITextField with default autocorrection.
    
    static NSString *const FXFormFieldTypeURL = @"url";
    
Like FXFormFieldTypeText, but with a keyboard type of UIKeyboardTypeURL, and no autocorrection.
    
    static NSString *const FXFormFieldTypeEmail = @"email";
    
Like FXFormFieldTypeText, but with a keyboard type of UIKeyboardTypeEmailAddress, and no autocorrection.
    
    static NSString *const FXFormFieldTypePassword = @"password";
    
Like FXFormFieldTypeText, but with secure text entry enabled, and no autocorrection.
    
    static NSString *const FXFormFieldTypeNumber = @"number";
    
Like FXFormFieldTypeText, but with a numeric keyboard, and input restricted to a valid number.
    
    static NSString *const FXFormFieldTypeInteger = @"integer";
    
Like FXFormFieldTypeNumber, but restricted to integer input.
    
    static NSString *const FXFormFieldTypeSwitch = @"switch";
    
A boolean value, set using a UISwitch control.
    
    static NSString *const FXFormFieldTypeStepper = @"stepper";
    
A numeric value, set using a UIStepper control.
    
    static NSString *const FXFormFieldTypeSlider = @"slider";
    
A numeric value, set using a UISlider control.
    
    static NSString *const FXFormFieldTypeCheckmark = @"checkmark";
    
A boolean value, toggled using a checkmark tick.

    static NSString *const FXFormFieldTypeDate = @"date";

A date value, selected using a UIDatePicker.


Form field options
----------------------

When you provide an options array for a form field, the field input will be presented as a list of options to tick. How this list of options is converted to the form value depends on the type of field:

If the field type matches the values in the options array, selecting the option will set the selected value directly, but that may not be what you want. For example, if you have a list of string, you may be more interested in the selected index than the value (which may have been localised and formatted for human consumption, not machine interpretation).

If the field type is numeric, and the options values are not numeric, it will be assumed that the field value should be set to the *index* of the selected item, instead of the value.


Cell configuration
-------------------

If you want to tweak some properties of the field cells, without subclassing them, you can actually set any cell property by keyPath, just by adding extra values to your field dictionary. For example, this code would turn the textLabel for the email field red:

    - (NSDictionary *)emailField
    {
        return @{@"textLabel.color": [UIColor redColor]};
    }
    
Cells are not recycled in the FXForm controller, so you don't need to worry about cleaning up any properties that you set in this way.
    

Custom cells
----------------

FXForms provides default cell implementations for all supported fields. You may wish to provide additional cell classes for custom field types, or even replace all of the FXForm cells with custom versions for your application.

There are two levels of customisation possible for cells. The simplest option is to subclass one of the existing FXFormCell classes, which all inherit from FXFormBaseCell. These cell classes contain a lot of logic for handling the various different field types, but expose the views and controls used, for easy customisation.

If you already have a base cell class and don't want to base your cells on FXFormBaseCell, you can create an FXForms-compatible cell from scratch by subclass UITableViewCell and adopting the FXFormFieldCell protocol.

Your custom cell must have a property called field, of type FXFormField. FXFormField is a wrapper class used to encapsulate the properties of a field, and also provides a way to set and get the associated form value (via the field.value virtual property). You cannot instantiate FXFormField wrappers directly, however the can be accessed and enumerated via methods on the FXFormController. FXFormField also provides the -performActionWithResponder:sender: that you can use to replicate the cascading action selector behavio of the default cells.

Once you have created your custom cell, you can use it as follows:

* If your cell is used only for a few specific fields, you can use the FXFormFieldCell property to use it for a particular form field
* If your cell is designed to handle a particular field type (or types), you can tell the formController to use your custom cell class for a particular type using the -registerCellClass:forFieldType: method of FXFormController.
* If you want to completely replace all cells with your own classes, use the -registerDefaultFieldCellClass: method of FXFormController. This replaces all default cell associations for all field types with your new cell class. You can then use -registerCellClass:forFieldType: to add additional cell classes for specific types.


Release Notes
--------------

Version 1.0 beta

- Initial release