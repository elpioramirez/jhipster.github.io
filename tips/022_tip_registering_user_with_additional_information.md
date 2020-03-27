---
layout: default
title: Registering a user with additional information
sitemap:
priority: 0.5
lastmod: 2017-02-15T22:30:00-00:00
---

# Registering a user with additional information

__Tip submitted by [@Paul-Etienne](https://github.com/Paul-Etienne)__

If we need to store more information concerning a user than what JHipster provides by default, a few tweaks are needed.

To illustrate this, we will display three extra values of different types: phone (as string), phoneType (enum), and photo (blob).

## Creating a new entity in a One to One relationship with JHI_User

The best way to add information in a One-To-One relationship with the User entity is by command line.

run in cmd:
```
jhipster entity UserExtra
```
respond questions as follow:
```
? Do you want to add a field to your entity? Yes
? What is the name of your field? phone
? What is the type of your field? String
? Do you want to add validation rules to your field? No
? Do you want to add a field to your entity? Yes
? What is the name of your field? type
? What is the type of your field? Enumeration (Java enum type)
? What is the class name of your enumeration? Type
? What are the values of your enumeration (separated by comma, no spaces)? Home,Mobile,Other
? Do you want to add validation rules to your field? No
? Do you want to add a field to your entity? Yes
? What is the name of your field? photo
? What is the type of your field? [BETA] Blob
? What is the content of the Blob field? An image
? Do you want to add validation rules to your field? No

? Do you want to add a relationship to another entity? Yes
? What is the name of the other entity? User
? What is the name of the relationship? user
? What is the type of the relationship? one-to-one
? Do you want to use JPA Derived Identifier - @MapsId? Yes
? Do you want to add any validation rules to this relationship? No

? Do you want to use separate service class for your business logic? Yes, generate a separate service interface and implementation
? Do you want to use a Data Transfer Object (DTO)? Yes, generate a DTO with MapStruct
? Do you want to add filtering? Dynamic filtering for the entities with JPA Static metamodel
? Is this entity read-only? No
? Do you want pagination on your entity? No

================= UserExtra =================
Fields
phone (String)
type (Type)
photo (byte[] image)

Relationships
user (User) one-to-one
```

This selection will create the correct One-To-One Relationship automatically including @MapsId annotation.

```
public class UserExtra implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    private Long id;

    @Column(name = "phone")
    private String phone;

    @OneToOne
    @MapsId
    private User user;
    ...

}
```

Note that the @GeneratedValue annotation on the id was removed.

# Updating Back-End

## Extending ManagedUserVM

Create class ManagedExtraVM in web/rest/vm has to be updated as well so that it holds the phone number sent by the client. The only thing to do here is adding the phone number attribute and its getter :

```
public class ManagedExtraVM extends ManagedUserVM {

    private String phone;
    
    private TypeId typeId;

    private byte[] photo;

    private String photoContentType;
    
    public ManagedExtraVM() {
    
    }
    ...
    
    // create setters and getters
    
    @Override
    public String toString() {
        return "ManagedExtraVM{" + super.toString() + "} ";
    }

}
```

## Updating the registerAccount() function from AccountResource

In directory web/rest/ the registerAccount() function now receives a ManagedExtraVM object that also contains the extra fields of the user. The only thing left to do is changing the class associated with the JHipster User from ManagedUserVM to ManagedExtraVM.

```
@PostMapping("/register")
@ResponseStatus(HttpStatus.CREATED)
public void registerAccount(@Valid @RequestBody ManagedExtraVM managedUserVM) {
    if (!checkPasswordLength(managedUserVM.getPassword())) {
        throw new InvalidPasswordException();
    }
    User user = userService.registerUser(managedUserVM, managedUserVM.getPassword());
    mailService.sendActivationEmail(user);
}
```

## Updating the createUser() function from UserService

Finally, we update the service layer function in  that saves the JHI_User to now save the UserExtra as well. Rather than updating the existing function, I suggest you create a new one with the additional parameter. This way, updating the test classes isn't necessary.

Do not forget to inject the UserExtra repositories :

```
private UserExtraRepository userExtraRepository;
...
public UserService(UserRepository userRepository, DatosUsuarioRepository extraRepository, PasswordEncoder passwordEncoder, ...) {
    this.userRepository = userRepository;
    this.extraRepository = extraRepository;
    this.passwordEncoder = passwordEncoder;
    ...
}
...
public User createUser(ManagedExtraVM userDTO, String password) {
    ...
    User newUser = new User();
    String encryptedPassword = passwordEncoder.encode(password);
    newUser.setLogin(userDTO.getLogin().toLowerCase());
    // new user gets initially a generated password
    newUser.setPassword(encryptedPassword);
    newUser.setFirstName(userDTO.getFirstName());
    newUser.setLastName(userDTO.getLastName());
    if (userDTO.getEmail() != null) {
        newUser.setEmail(userDTO.getEmail().toLowerCase());
    }
    newUser.setImageUrl(userDTO.getImageUrl());
    newUser.setLangKey(userDTO.getLangKey());
    // new user is not active
    newUser.setActivated(false);
    // new user gets registration key
    newUser.setActivationKey(RandomUtil.generateActivationKey());
    Set<Authority> authorities = new HashSet<>();
    authorityRepository.findById(AuthoritiesConstants.USER).ifPresent(authorities::add);
    newUser.setAuthorities(authorities);
    userRepository.save(newUser);
    log.debug("Created Information for User: {}", newUser);

    // Create and save the UserExtra entity
    UserExtra newUserExtra = new UserExtra();
    newUserExtra.setUser(newUser);
    newUserExtra.setPhone(userDTO.getPhone());
    newExtra.setTipoId(userDTO.getTipoId());
    newExtra.setFotoContentType(userDTO.getFotoContentType());
    newExtra.setFoto(userDTO.getFoto());
    userExtraRepository.save(newUserExtra);
    userExtraSearchRepository.save(newUserExtra);
    log.debug("Created Information for UserExtra: {}", newUserExtra);

    return newUser;
}
```

And it's done for Backend!

# Updating Front-End (Angular)
## Update user model
Update user.model.ts in webapp/app/core/user/
```
import { TipoId } from 'app/shared/model/enumerations/tipo-id.model';

export interface IUser {
  ...
  phone?: string;
  typeId?: TipoId;
  photoContentType?: string;
  photo?: any;
}

export class User implements IUser {
  constructor(
    ...
    public phone?: string,
    public typeId?: TipoId,
    public photoContentType?: string,
    public photo?: any
  ) {}
}
```
## Updating the register HTML page

Now that an entity exists to store the fields, we need to add an input in the register form to ask for the user's extra fields.

Nothing easier than that, just update webapp/app/account/register/register.component.html to add an input field bound to the variable already used to store the basic information (vm.registerAccount) :

```
<div class="form-group">
    <label class="form-control-label" jhiTranslate="*.userExtra.phone" for="field_phone">Phone</label>
    <input type="text" class="form-control" name="phone" id="field_phone"
           formControlName="phone"/>
</div>
<div class="form-group">
    <label class="form-control-label" jhiTranslate="*.userExtra.typeId" for="field_typeId">Type Id</label>
    <select class="form-control" name="typeId" formControlName="typeId" id="field_typeId">
        <option value="OFICIAL">{{ '*.typeId.OFICIAL' | translate }}</option>
        <option value="PASAPORTE">{{ '*.typeId.PASAPORTE' | translate }}</option>
        <option value="OTRO">{{ '*.typeId.OTRO' | translate }}</option>
    </select>
</div>
<div class="form-group">
    <label class="form-control-label" jhiTranslate="*.userExtra.photo" for="field_photo">Photo</label>
    <div>
        <img [src]="'data:' + registerForm.get('photoContentType')!.value + ';base64,' + registerForm.get('photo')!.value" 
            style="max-height: 100px;" *ngIf="registerForm.get('photo')!.value" alt="userExtra image"/>
        <div *ngIf="registerForm.get('photo')!.value" class="form-text text-danger clearfix">
            <span class="pull-left">{{ registerForm.get('photoContentType')!.value }}, {{ byteSize(registerForm.get('photo')!.value) }}
            </span>
            <button type="button" (click)="clearInputImage('photo', 'photoContentType', 'file_photo')" 
                class="btn btn-secondary btn-xs pull-right">
                <fa-icon icon="times"></fa-icon>
            </button>
        </div>
        <input type="file" id="file_photo" (change)="setFileData($event, 'photo', true)" 
            accept="image/*" jhiTranslate="entity.action.addimage"/>
    </div>
    <input type="hidden" class="form-control" name="photo" id="field_photo"
           formControlName="photo"/>
    <input type="hidden" class="form-control" name="photoContentType" id="field_photoContentType"
           formControlName="photoContentType" />
</div>
```
## Updating the register Component

```
...
import { JhiDataUtils, JhiFileLoadError, JhiEventManager, JhiEventWithContent, JhiLanguageService  } from 'ng-jhipster';
import { AlertError } from 'app/shared/alert/alert-error.model';
...
  registerForm = this.fb.group({
    ...
    phone: [],
    typeId: [],
    photo: [],
    photoContentType: []
  });
  
  constructor(
    ...
    protected dataUtils: JhiDataUtils,
    protected eventManager: JhiEventManager,
    protected elementRef: ElementRef,
  ) {}
  ...
  register(): void {
      ...
      const phoe = this.registerForm.get(['phone'])!.value;
      const typeId = this.registerForm.get(['typeId'])!.value;
      const photoContentType = this.registerForm.get(['photoContentType'])!.value;
      const photo = this.registerForm.get(['photo'])!.value;
      this.registerService.save({ ..., phone, typeId, photoContentType, photo }).subscribe(
        () => (this.success = true),
        response => this.processError(response)
      );
    }
  }
  
  ...
  //for photo validations
  
  byteSize(base64String: string): string {
    return this.dataUtils.byteSize(base64String);
  }

  clearInputImage(field: string, fieldContentType: string, idInput: string): void {
    this.registerForm.patchValue({
      [field]: null,
      [fieldContentType]: null
    });
    if (this.elementRef && idInput && this.elementRef.nativeElement.querySelector('#' + idInput)) {
      this.elementRef.nativeElement.querySelector('#' + idInput).value = null;
    }
  }

  setFileData(event: Event, field: string, isImage: boolean): void {
    this.dataUtils.loadFileToForm(event, this.registerForm, field, isImage).subscribe(null, (err: JhiFileLoadError) => {
      this.eventManager.broadcast(
        new JhiEventWithContent<AlertError>('userApp.error', { ...err, key: 'error.file.' + err.key })
      );
    });
  }

  openFile(contentType: string, base64String: string): void {
    this.dataUtils.openFile(contentType, base64String);
  }

```
