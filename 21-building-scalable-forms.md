# Chapter 21: Building Scalable Forms

## Introduction

Junior developers build forms with controlled inputs and manual validation, creating performance issues and messy code. Senior developers use form libraries like React Hook Form with schema validation, building performant, type-safe forms that scale from simple logins to complex multi-step wizards.

## Learning Objectives

- Master React Hook Form for performant forms
- Implement schema validation with Zod
- Build dynamic form fields
- Create multi-step form wizards
- Handle file uploads in forms
- Implement form arrays and nested objects
- Add conditional field logic
- Optimize form performance

## 21.1 React Hook Form Fundamentals

### Basic Form Setup

```typescript
// Install: npm install react-hook-form zod @hookform/resolvers

import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

// Define schema
const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  rememberMe: z.boolean().optional(),
});

type LoginFormData = z.infer<typeof loginSchema>;

function LoginForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
    defaultValues: {
      email: '',
      password: '',
      rememberMe: false,
    },
  });

  const onSubmit = async (data: LoginFormData) => {
    try {
      await authApi.login(data);
      toast.success('Login successful');
    } catch (error) {
      toast.error('Login failed');
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          {...register('email')}
          aria-invalid={errors.email ? 'true' : 'false'}
        />
        {errors.email && (
          <span role="alert" className="error">
            {errors.email.message}
          </span>
        )}
      </div>

      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          {...register('password')}
          aria-invalid={errors.password ? 'true' : 'false'}
        />
        {errors.password && (
          <span role="alert" className="error">
            {errors.password.message}
          </span>
        )}
      </div>

      <div>
        <label>
          <input type="checkbox" {...register('rememberMe')} />
          Remember me
        </label>
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
}
```

### Reusable Form Components

```typescript
// components/Form/FormField.tsx
interface FormFieldProps {
  name: string;
  label: string;
  type?: string;
  register: any;
  error?: FieldError;
  placeholder?: string;
  required?: boolean;
}

export function FormField({
  name,
  label,
  type = 'text',
  register,
  error,
  placeholder,
  required,
}: FormFieldProps) {
  return (
    <div className="form-field">
      <label htmlFor={name}>
        {label}
        {required && <span className="required">*</span>}
      </label>

      <input
        id={name}
        type={type}
        {...register(name)}
        placeholder={placeholder}
        aria-invalid={error ? 'true' : 'false'}
        aria-describedby={error ? `${name}-error` : undefined}
      />

      {error && (
        <span id={`${name}-error`} role="alert" className="error">
          {error.message}
        </span>
      )}
    </div>
  );
}

// Usage
function UserForm() {
  const { register, handleSubmit, formState: { errors } } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <FormField
        name="firstName"
        label="First Name"
        register={register}
        error={errors.firstName}
        required
      />

      <FormField
        name="email"
        label="Email"
        type="email"
        register={register}
        error={errors.email}
        required
      />
    </form>
  );
}
```

## 21.2 Advanced Validation Patterns

### Complex Schema Validation

```typescript
const registrationSchema = z.object({
  username: z.string()
    .min(3, 'Username must be at least 3 characters')
    .max(20, 'Username must be less than 20 characters')
    .regex(/^[a-zA-Z0-9_]+$/, 'Username can only contain letters, numbers, and underscores'),

  email: z.string()
    .email('Invalid email address')
    .refine(
      async (email) => {
        // Async validation - check if email is available
        const available = await checkEmailAvailability(email);
        return available;
      },
      { message: 'Email already in use' }
    ),

  password: z.string()
    .min(8, 'Password must be at least 8 characters')
    .regex(/[A-Z]/, 'Password must contain at least one uppercase letter')
    .regex(/[a-z]/, 'Password must contain at least one lowercase letter')
    .regex(/[0-9]/, 'Password must contain at least one number')
    .regex(/[^A-Za-z0-9]/, 'Password must contain at least one special character'),

  confirmPassword: z.string(),

  age: z.coerce.number()
    .min(18, 'Must be at least 18 years old')
    .max(120, 'Invalid age'),

  terms: z.literal(true, {
    errorMap: () => ({ message: 'You must accept the terms and conditions' })
  }),

  phoneNumber: z.string()
    .regex(/^\+?[1-9]\d{1,14}$/, 'Invalid phone number')
    .optional(),

  website: z.string().url('Invalid URL').optional().or(z.literal('')),
}).refine((data) => data.password === data.confirmPassword, {
  message: 'Passwords do not match',
  path: ['confirmPassword'],
});

type RegistrationFormData = z.infer<typeof registrationSchema>;
```

### Conditional Validation

```typescript
const profileSchema = z.object({
  accountType: z.enum(['personal', 'business']),

  // Personal fields
  firstName: z.string().optional(),
  lastName: z.string().optional(),

  // Business fields
  companyName: z.string().optional(),
  taxId: z.string().optional(),

  email: z.string().email(),
}).refine(
  (data) => {
    if (data.accountType === 'personal') {
      return !!data.firstName && !!data.lastName;
    }
    return true;
  },
  {
    message: 'First name and last name are required for personal accounts',
    path: ['firstName'],
  }
).refine(
  (data) => {
    if (data.accountType === 'business') {
      return !!data.companyName && !!data.taxId;
    }
    return true;
  },
  {
    message: 'Company name and tax ID are required for business accounts',
    path: ['companyName'],
  }
);

function ProfileForm() {
  const { register, watch, formState: { errors } } = useForm({
    resolver: zodResolver(profileSchema),
  });

  const accountType = watch('accountType');

  return (
    <form>
      <select {...register('accountType')}>
        <option value="personal">Personal</option>
        <option value="business">Business</option>
      </select>

      {accountType === 'personal' && (
        <>
          <FormField name="firstName" label="First Name" register={register} error={errors.firstName} />
          <FormField name="lastName" label="Last Name" register={register} error={errors.lastName} />
        </>
      )}

      {accountType === 'business' && (
        <>
          <FormField name="companyName" label="Company Name" register={register} error={errors.companyName} />
          <FormField name="taxId" label="Tax ID" register={register} error={errors.taxId} />
        </>
      )}
    </form>
  );
}
```

## 21.3 Dynamic Form Fields

### Field Arrays

```typescript
const schema = z.object({
  name: z.string().min(1, 'Name is required'),
  emails: z.array(z.object({
    email: z.string().email('Invalid email'),
    type: z.enum(['work', 'personal']),
  })).min(1, 'At least one email is required'),
});

type FormData = z.infer<typeof schema>;

function ContactForm() {
  const { register, control, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
    defaultValues: {
      name: '',
      emails: [{ email: '', type: 'personal' }],
    },
  });

  const { fields, append, remove } = useFieldArray({
    control,
    name: 'emails',
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <FormField
        name="name"
        label="Name"
        register={register}
        error={errors.name}
      />

      <div>
        <h3>Emails</h3>
        {fields.map((field, index) => (
          <div key={field.id} className="field-array-item">
            <input
              {...register(`emails.${index}.email`)}
              placeholder="Email"
            />
            {errors.emails?.[index]?.email && (
              <span className="error">
                {errors.emails[index]?.email?.message}
              </span>
            )}

            <select {...register(`emails.${index}.type`)}>
              <option value="personal">Personal</option>
              <option value="work">Work</option>
            </select>

            {fields.length > 1 && (
              <button type="button" onClick={() => remove(index)}>
                Remove
              </button>
            )}
          </div>
        ))}

        <button
          type="button"
          onClick={() => append({ email: '', type: 'personal' })}
        >
          Add Email
        </button>
      </div>

      <button type="submit">Submit</button>
    </form>
  );
}
```

### Nested Objects

```typescript
const addressSchema = z.object({
  street: z.string().min(1, 'Street is required'),
  city: z.string().min(1, 'City is required'),
  state: z.string().min(2, 'State is required'),
  zipCode: z.string().regex(/^\d{5}(-\d{4})?$/, 'Invalid ZIP code'),
});

const userSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  shippingAddress: addressSchema,
  billingAddress: addressSchema.optional(),
  sameAsBilling: z.boolean(),
});

type UserFormData = z.infer<typeof userSchema>;

function AddressForm() {
  const { register, watch, setValue, formState: { errors } } = useForm<UserFormData>({
    resolver: zodResolver(userSchema),
  });

  const sameAsBilling = watch('sameAsBilling');
  const shippingAddress = watch('shippingAddress');

  useEffect(() => {
    if (sameAsBilling) {
      setValue('billingAddress', shippingAddress);
    }
  }, [sameAsBilling, shippingAddress, setValue]);

  return (
    <form>
      <h3>Shipping Address</h3>
      <FormField name="shippingAddress.street" label="Street" register={register} error={errors.shippingAddress?.street} />
      <FormField name="shippingAddress.city" label="City" register={register} error={errors.shippingAddress?.city} />
      <FormField name="shippingAddress.state" label="State" register={register} error={errors.shippingAddress?.state} />
      <FormField name="shippingAddress.zipCode" label="ZIP Code" register={register} error={errors.shippingAddress?.zipCode} />

      <label>
        <input type="checkbox" {...register('sameAsBilling')} />
        Same as shipping address
      </label>

      {!sameAsBilling && (
        <>
          <h3>Billing Address</h3>
          <FormField name="billingAddress.street" label="Street" register={register} error={errors.billingAddress?.street} />
          <FormField name="billingAddress.city" label="City" register={register} error={errors.billingAddress?.city} />
          <FormField name="billingAddress.state" label="State" register={register} error={errors.billingAddress?.state} />
          <FormField name="billingAddress.zipCode" label="ZIP Code" register={register} error={errors.billingAddress?.zipCode} />
        </>
      )}
    </form>
  );
}
```

## 21.4 Multi-Step Forms

### Wizard with State Management

```typescript
// hooks/useMultiStepForm.ts
export function useMultiStepForm<T extends z.ZodType>(
  steps: { schema: T; component: React.ComponentType<any> }[]
) {
  const [currentStep, setCurrentStep] = useState(0);
  const [formData, setFormData] = useState<Partial<z.infer<T>>>({});

  const isFirstStep = currentStep === 0;
  const isLastStep = currentStep === steps.length - 1;

  const goToNext = (data: Partial<z.infer<T>>) => {
    setFormData((prev) => ({ ...prev, ...data }));
    setCurrentStep((prev) => Math.min(prev + 1, steps.length - 1));
  };

  const goToPrevious = () => {
    setCurrentStep((prev) => Math.max(prev - 1, 0));
  };

  const goToStep = (step: number) => {
    setCurrentStep(Math.max(0, Math.min(step, steps.length - 1)));
  };

  return {
    currentStep,
    isFirstStep,
    isLastStep,
    formData,
    goToNext,
    goToPrevious,
    goToStep,
    currentSchema: steps[currentStep].schema,
    CurrentStepComponent: steps[currentStep].component,
  };
}

// components/RegistrationWizard.tsx
const step1Schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
});

const step2Schema = z.object({
  firstName: z.string().min(1),
  lastName: z.string().min(1),
});

const step3Schema = z.object({
  bio: z.string().max(500).optional(),
  website: z.string().url().optional(),
});

function Step1({ data, onNext }: any) {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(step1Schema),
    defaultValues: data,
  });

  return (
    <form onSubmit={handleSubmit(onNext)}>
      <h2>Account Information</h2>
      <FormField name="email" label="Email" type="email" register={register} error={errors.email} />
      <FormField name="password" label="Password" type="password" register={register} error={errors.password} />
      <button type="submit">Next</button>
    </form>
  );
}

function Step2({ data, onNext, onBack }: any) {
  const { register, handleSubmit, formState: { errors } } = useForm({
    resolver: zodResolver(step2Schema),
    defaultValues: data,
  });

  return (
    <form onSubmit={handleSubmit(onNext)}>
      <h2>Personal Information</h2>
      <FormField name="firstName" label="First Name" register={register} error={errors.firstName} />
      <FormField name="lastName" label="Last Name" register={register} error={errors.lastName} />
      <button type="button" onClick={onBack}>Back</button>
      <button type="submit">Next</button>
    </form>
  );
}

function Step3({ data, onSubmit, onBack }: any) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    resolver: zodResolver(step3Schema),
    defaultValues: data,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <h2>Profile Details</h2>
      <div>
        <label htmlFor="bio">Bio</label>
        <textarea id="bio" {...register('bio')} rows={4} />
        {errors.bio && <span className="error">{errors.bio.message}</span>}
      </div>
      <FormField name="website" label="Website" register={register} error={errors.website} />
      <button type="button" onClick={onBack}>Back</button>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Submitting...' : 'Complete Registration'}
      </button>
    </form>
  );
}

function RegistrationWizard() {
  const steps = [
    { schema: step1Schema, component: Step1 },
    { schema: step2Schema, component: Step2 },
    { schema: step3Schema, component: Step3 },
  ];

  const {
    currentStep,
    isFirstStep,
    isLastStep,
    formData,
    goToNext,
    goToPrevious,
    CurrentStepComponent,
  } = useMultiStepForm(steps);

  const handleFinalSubmit = async (data: any) => {
    const completeData = { ...formData, ...data };
    await userApi.register(completeData);
    toast.success('Registration complete!');
  };

  return (
    <div className="wizard">
      <div className="wizard-progress">
        {steps.map((_, index) => (
          <div
            key={index}
            className={`step ${index === currentStep ? 'active' : ''} ${
              index < currentStep ? 'completed' : ''
            }`}
          >
            Step {index + 1}
          </div>
        ))}
      </div>

      <CurrentStepComponent
        data={formData}
        onNext={goToNext}
        onBack={goToPrevious}
        onSubmit={handleFinalSubmit}
      />
    </div>
  );
}
```

## 21.5 File Uploads

### Single File Upload

```typescript
const fileSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  avatar: z.instanceof(FileList)
    .refine((files) => files.length > 0, 'File is required')
    .refine(
      (files) => files[0]?.size <= 5 * 1024 * 1024,
      'File size must be less than 5MB'
    )
    .refine(
      (files) => ['image/jpeg', 'image/png', 'image/webp'].includes(files[0]?.type),
      'Only .jpg, .png, and .webp files are accepted'
    ),
});

type FileFormData = z.infer<typeof fileSchema>;

function FileUploadForm() {
  const { register, handleSubmit, watch, formState: { errors } } = useForm<FileFormData>({
    resolver: zodResolver(fileSchema),
  });

  const avatarFiles = watch('avatar');
  const [preview, setPreview] = useState<string | null>(null);

  useEffect(() => {
    if (avatarFiles && avatarFiles.length > 0) {
      const file = avatarFiles[0];
      const reader = new FileReader();
      reader.onloadend = () => {
        setPreview(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  }, [avatarFiles]);

  const onSubmit = async (data: FileFormData) => {
    const formData = new FormData();
    formData.append('name', data.name);
    formData.append('avatar', data.avatar[0]);

    await userApi.uploadAvatar(formData);
    toast.success('Avatar uploaded');
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <FormField name="name" label="Name" register={register} error={errors.name} />

      <div>
        <label htmlFor="avatar">Avatar</label>
        <input
          id="avatar"
          type="file"
          accept="image/jpeg,image/png,image/webp"
          {...register('avatar')}
        />
        {errors.avatar && (
          <span className="error">{errors.avatar.message as string}</span>
        )}

        {preview && (
          <div className="preview">
            <img src={preview} alt="Preview" width={150} height={150} />
          </div>
        )}
      </div>

      <button type="submit">Upload</button>
    </form>
  );
}
```

### Multiple File Upload with Drag and Drop

```typescript
// components/FileDropzone.tsx
import { useDropzone } from 'react-dropzone';

interface FileDropzoneProps {
  onFilesAccepted: (files: File[]) => void;
  maxFiles?: number;
  maxSize?: number;
  accept?: Record<string, string[]>;
}

export function FileDropzone({
  onFilesAccepted,
  maxFiles = 5,
  maxSize = 5 * 1024 * 1024,
  accept = { 'image/*': ['.png', '.jpg', '.jpeg', '.webp'] },
}: FileDropzoneProps) {
  const {
    getRootProps,
    getInputProps,
    isDragActive,
    fileRejections,
  } = useDropzone({
    onDrop: onFilesAccepted,
    maxFiles,
    maxSize,
    accept,
  });

  return (
    <div
      {...getRootProps()}
      className={`dropzone ${isDragActive ? 'active' : ''}`}
    >
      <input {...getInputProps()} />
      {isDragActive ? (
        <p>Drop files here...</p>
      ) : (
        <p>Drag and drop files here, or click to select</p>
      )}

      {fileRejections.length > 0 && (
        <div className="rejections">
          {fileRejections.map(({ file, errors }) => (
            <div key={file.name}>
              {file.name}: {errors.map((e) => e.message).join(', ')}
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

// Usage
function GalleryUploadForm() {
  const [files, setFiles] = useState<File[]>([]);

  const handleFilesAccepted = (acceptedFiles: File[]) => {
    setFiles((prev) => [...prev, ...acceptedFiles]);
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();

    const formData = new FormData();
    files.forEach((file) => {
      formData.append('images', file);
    });

    await galleryApi.upload(formData);
    toast.success('Images uploaded');
  };

  return (
    <form onSubmit={handleSubmit}>
      <FileDropzone onFilesAccepted={handleFilesAccepted} />

      {files.length > 0 && (
        <div className="file-list">
          {files.map((file, index) => (
            <div key={index} className="file-item">
              <span>{file.name}</span>
              <button
                type="button"
                onClick={() => setFiles((prev) => prev.filter((_, i) => i !== index))}
              >
                Remove
              </button>
            </div>
          ))}
        </div>
      )}

      <button type="submit" disabled={files.length === 0}>
        Upload {files.length} file(s)
      </button>
    </form>
  );
}
```

## 21.6 Form Performance Optimization

### Uncontrolled Components

```typescript
// ✅ Good: Uncontrolled (React Hook Form default)
// Only re-renders on submit or validation
function OptimizedForm() {
  const { register, handleSubmit } = useForm();

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      <input {...register('email')} />
      {/* Component doesn't re-render on each keystroke */}
    </form>
  );
}

// ❌ Bad: Controlled with useState
// Re-renders on every keystroke
function SlowForm() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');

  return (
    <form>
      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <input
        value={email}
        onChange={(e) => setEmail(e.target.value)}
      />
      {/* Every keystroke causes re-render */}
    </form>
  );
}
```

### Debounced Validation

```typescript
// hooks/useDebouncedValidation.ts
export function useDebouncedValidation(
  fieldName: string,
  validator: (value: any) => Promise<boolean>,
  delay: number = 500
) {
  const { setError, clearErrors } = useFormContext();

  const debouncedValidate = useDeferredValue(
    useCallback(
      async (value: any) => {
        const isValid = await validator(value);
        if (!isValid) {
          setError(fieldName, {
            type: 'manual',
            message: 'Validation failed',
          });
        } else {
          clearErrors(fieldName);
        }
      },
      [fieldName, validator, setError, clearErrors]
    ),
    delay
  );

  return debouncedValidate;
}

// Usage
function UsernameField() {
  const { register, watch } = useForm();
  const username = watch('username');

  const validateUsername = async (value: string) => {
    if (value.length < 3) return true;
    const available = await checkUsernameAvailability(value);
    return available;
  };

  const debouncedValidate = useDebouncedValidation('username', validateUsername);

  useEffect(() => {
    if (username) {
      debouncedValidate(username);
    }
  }, [username, debouncedValidate]);

  return <input {...register('username')} />;
}
```

## Real-World Scenario: Building a Job Application Form

### The Challenge

Build a complex job application form:
- Personal information
- Work experience (dynamic array)
- Education history (dynamic array)
- File uploads (resume, cover letter)
- Multi-step wizard
- Draft saving
- Validation

### Senior Approach

```typescript
// Complete implementation with all best practices

const applicationSchema = z.object({
  // Step 1: Personal Info
  personalInfo: z.object({
    firstName: z.string().min(1),
    lastName: z.string().min(1),
    email: z.string().email(),
    phone: z.string().regex(/^\+?[1-9]\d{1,14}$/),
  }),

  // Step 2: Work Experience
  workExperience: z.array(z.object({
    company: z.string().min(1),
    position: z.string().min(1),
    startDate: z.date(),
    endDate: z.date().optional(),
    current: z.boolean(),
    description: z.string().max(500),
  })).min(1),

  // Step 3: Education
  education: z.array(z.object({
    institution: z.string().min(1),
    degree: z.string().min(1),
    graduationDate: z.date(),
  })),

  // Step 4: Documents
  resume: z.instanceof(FileList).refine((files) => files.length > 0),
  coverLetter: z.instanceof(FileList).optional(),
});

// Features:
// - Multi-step wizard with progress
// - Field arrays for experience/education
// - File uploads with previews
// - Draft auto-save to localStorage
// - Validation per step
// - Type-safe throughout
// - Optimized performance
```

## Chapter Exercise: Build Complex Form

Create a comprehensive form system:

**Requirements:**
1. Multi-step wizard (3+ steps)
2. Dynamic field arrays
3. File upload with validation
4. Conditional fields
5. Schema validation with Zod
6. Auto-save drafts
7. Performance optimized
8. Accessible (ARIA labels, error announcements)

**Bonus:**
- Progress persistence across sessions
- Resume from any step
- Field-level async validation

## Review Checklist

- [ ] React Hook Form configured
- [ ] Zod schema validation
- [ ] Field arrays working
- [ ] Multi-step wizard
- [ ] File uploads handled
- [ ] Conditional fields
- [ ] Error messages clear
- [ ] Performance optimized
- [ ] Accessible
- [ ] Type-safe

## Key Takeaways

1. **Use React Hook Form** - Better performance than controlled forms
2. **Validate with Zod** - Type-safe schema validation
3. **Field arrays for dynamic lists** - Built-in support
4. **Optimize with uncontrolled inputs** - Fewer re-renders
5. **Multi-step forms improve UX** - Break complex forms down
6. **File uploads need validation** - Size, type, count
7. **Accessibility matters** - ARIA labels and error announcements

## Further Reading

- React Hook Form documentation
- Zod documentation
- Form accessibility guidelines (WCAG)
- Multi-step form UX patterns

## Next Chapter

[Chapter 22: Real-Time Features](./22-real-time-features.md)
