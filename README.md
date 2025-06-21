import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, getDoc, collection, addDoc, serverTimestamp } from 'firebase/firestore';

// Main App component for the Job Analysis Questionnaire
function App() {
  const [formData, setFormData] = useState({
    yourName: '',
    employeeId: '',
    yourJobTitle: '',
    yearsInPosition: '',
    monthsInPosition: '',
    workPhoneNumber: '',
    supervisorName: '',
    supervisorTitle: '',
    divisionCollege: '',
    department: '',
    jobCode: '',
    generalPurpose: '',
    responsibilities: Array(9).fill({ description: '', percentage: '' }),
    educationRequirement: '',
    experienceType: '',
    experienceAmount: '',
    skillsCertifications: '',
    supervisoryNature: '',
    directReportsCount: '',
    directReportTitles: Array(3).fill({ title: '', gradeLevel: '', numPositions: '' }),
    indirectReportsCount: '',
    functionalSupervision: '',
    orgChartSupervisor: '',
    orgChartYourPosition: '',
    orgChartOtherJobsImmediateSupervisor: ['', ''],
    orgChartDirectReports: ['', '', '', '', '', ''],
    physicalDemands: {},
    environmentalConditions: {},
    physicalStrength: ''
  });

  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);
  const [submissionStatus, setSubmissionStatus] = useState('');
  const [showConfirmation, setShowConfirmation] = useState(false);
  const [loading, setLoading] = useState(true);
  const formRef = useRef(null);

  // Initialize Firebase and set up auth listener
  useEffect(() => {
    try {
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

      if (Object.keys(firebaseConfig).length === 0) {
        console.error("Firebase config is not provided. Cannot initialize Firebase.");
        setLoading(false);
        return;
      }

      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const authInstance = getAuth(app);

      setDb(firestore);
      setAuth(authInstance);

      onAuthStateChanged(authInstance, async (user) => {
        if (user) {
          setUserId(user.uid);
        } else {
          try {
            const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;
            if (initialAuthToken) {
              await signInWithCustomToken(authInstance, initialAuthToken);
            } else {
              await signInAnonymously(authInstance);
            }
            setUserId(authInstance.currentUser?.uid || crypto.randomUUID());
          } catch (error) {
            console.error("Firebase authentication error:", error);
            setUserId(crypto.randomUUID()); // Fallback to a random ID if auth fails
          }
        }
        setIsAuthReady(true);
        setLoading(false);
      });
    } catch (error) {
      console.error("Error initializing Firebase:", error);
      setLoading(false);
    }
  }, []);

  // Handle form field changes, including nested responsibility fields
  const handleChange = (e) => {
    const { name, value } = e.target;
    if (name.startsWith('responsibilityDescription') || name.startsWith('responsibilityPercentage')) {
      const index = parseInt(name.replace('responsibilityDescription', '').replace('responsibilityPercentage', ''), 10);
      const field = name.includes('Description') ? 'description' : 'percentage';
      const newResponsibilities = [...formData.responsibilities];
      newResponsibilities[index] = { ...newResponsibilities[index], [field]: value };
      setFormData({ ...formData, responsibilities: newResponsibilities });
    } else if (name.startsWith('directReportTitle') || name.startsWith('directReportGrade') || name.startsWith('directReportNum')) {
      const index = parseInt(name.replace('directReportTitle', '').replace('directReportGrade', '').replace('directReportNum', ''), 10);
      const field = name.includes('Title') ? 'title' : name.includes('Grade') ? 'gradeLevel' : 'numPositions';
      const newDirectReports = [...formData.directReportTitles];
      newDirectReports[index] = { ...newDirectReports[index], [field]: value };
      setFormData({ ...formData, directReportTitles: newDirectReports });
    } else if (name.startsWith('orgChartOtherJobsImmediateSupervisor')) {
      const index = parseInt(name.replace('orgChartOtherJobsImmediateSupervisor', ''), 10);
      const newOtherJobs = [...formData.orgChartOtherJobsImmediateSupervisor];
      newOtherJobs[index] = value;
      setFormData({ ...formData, orgChartOtherJobsImmediateSupervisor: newOtherJobs });
    } else if (name.startsWith('orgChartDirectReports')) {
      const index = parseInt(name.replace('orgChartDirectReports', ''), 10);
      const newDirectReports = [...formData.orgChartDirectReports];
      newDirectReports[index] = value;
      setFormData({ ...formData, orgChartDirectReports: newDirectReports });
    } else {
      setFormData({ ...formData, [name]: value });
    }
  };

  // Handle checkbox changes for physical demands and environmental conditions
  const handleCheckboxChange = (section, e) => {
    const { name, value, type, checked } = e.target;
    if (type === 'radio') {
      setFormData(prev => ({
        ...prev,
        [section]: {
          ...prev[section],
          [name]: checked ? value : '' // Only set if checked for radio
        }
      }));
    } else {
      setFormData(prev => ({
        ...prev,
        [section]: {
          ...prev[section],
          [name]: checked ? value : '' // Store value if checked, empty if unchecked
        }
      }));
    }
  };


  // Submission handler
  const handleSubmit = async (e) => {
    e.preventDefault();
    if (!db || !userId) {
      setSubmissionStatus("Error: Firebase not initialized or user not authenticated. Please try again.");
      return;
    }

    setShowConfirmation(true);
  };

  const confirmSubmit = async () => {
    setShowConfirmation(false);
    setSubmissionStatus("Submitting...");

    try {
      // Data to be saved, including timestamp
      const dataToSave = {
        ...formData,
        timestamp: serverTimestamp(),
        userId: userId, // Store the user ID with the submission
        appId: typeof __app_id !== 'undefined' ? __app_id : 'default-app-id' // Store the app ID
      };

      // Define collection path based on Firebase security rules
      const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
      const collectionPath = `artifacts/${appId}/users/${userId}/jobQuestionnaires`;

      // Add a new document to the 'jobQuestionnaires' collection
      await addDoc(collection(db, collectionPath), dataToSave);

      setSubmissionStatus("Form submitted successfully! Your responses have been saved.");
      // Optionally reset form after successful submission
      // setFormData({ /* initial state */ });
    } catch (error) {
      console.error("Error submitting document: ", error);
      setSubmissionStatus(`Error submitting form: ${error.message}`);
    }
  };

  const cancelSubmit = () => {
    setShowConfirmation(false);
    setSubmissionStatus("Submission cancelled.");
  };

  // Show a loading indicator while Firebase initializes
  if (loading) {
    return (
      <div className="flex items-center justify-center min-h-screen bg-gray-100">
        <div className="text-xl font-semibold text-gray-700">Loading form...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-6 font-inter flex justify-center">
      <div className="bg-white p-8 rounded-xl shadow-2xl max-w-4xl w-full">
        <h1 className="text-4xl font-extrabold text-center text-gray-900 mb-6 border-b-4 border-indigo-600 pb-3">
          VigilanceIQ Job Analysis Questionnaire
        </h1>
        <p className="text-center text-lg text-gray-700 mb-8">
          Please complete this questionnaire to provide current information on your job duties and responsibilities.
        </p>

        {userId && (
          <p className="text-center text-sm text-gray-500 mb-4">
            User ID: <span className="font-mono bg-gray-100 px-2 py-1 rounded-md">{userId}</span>
          </p>
        )}

        <form onSubmit={handleSubmit} className="space-y-8" ref={formRef}>
          {/* Section A: Employee Data */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">A. EMPLOYEE DATA</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
              <div>
                <label htmlFor="yourName" className="block text-sm font-medium text-gray-700 mb-1">Your Name:</label>
                <input
                  type="text"
                  id="yourName"
                  name="yourName"
                  value={formData.yourName}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                  required
                />
              </div>
              <div>
                <label htmlFor="employeeId" className="block text-sm font-medium text-gray-700 mb-1">Employee ID:</label>
                <input
                  type="text"
                  id="employeeId"
                  name="employeeId"
                  value={formData.employeeId}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                  required
                />
              </div>
              <div>
                <label htmlFor="yourJobTitle" className="block text-sm font-medium text-gray-700 mb-1">Your Job Title:</label>
                <input
                  type="text"
                  id="yourJobTitle"
                  name="yourJobTitle"
                  value={formData.yourJobTitle}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                  required
                />
              </div>
              <div className="flex space-x-4">
                <div className="flex-1">
                  <label htmlFor="yearsInPosition" className="block text-sm font-medium text-gray-700 mb-1">Years in current position:</label>
                  <input
                    type="number"
                    id="yearsInPosition"
                    name="yearsInPosition"
                    value={formData.yearsInPosition}
                    onChange={handleChange}
                    className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                    min="0"
                  />
                </div>
                <div className="flex-1">
                  <label htmlFor="monthsInPosition" className="block text-sm font-medium text-gray-700 mb-1">Months:</label>
                  <input
                    type="number"
                    id="monthsInPosition"
                    name="monthsInPosition"
                    value={formData.monthsInPosition}
                    onChange={handleChange}
                    className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                    min="0"
                    max="11"
                  />
                </div>
              </div>
              <div>
                <label htmlFor="workPhoneNumber" className="block text-sm font-medium text-gray-700 mb-1">Work Telephone Number:</label>
                <input
                  type="tel"
                  id="workPhoneNumber"
                  name="workPhoneNumber"
                  value={formData.workPhoneNumber}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
              <div>
                <label htmlFor="supervisorName" className="block text-sm font-medium text-gray-700 mb-1">Supervisor's Name:</label>
                <input
                  type="text"
                  id="supervisorName"
                  name="supervisorName"
                  value={formData.supervisorName}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
              <div>
                <label htmlFor="supervisorTitle" className="block text-sm font-medium text-gray-700 mb-1">Supervisor's Title:</label>
                <input
                  type="text"
                  id="supervisorTitle"
                  name="supervisorTitle"
                  value={formData.supervisorTitle}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
            </div>
          </section>

          {/* Section B: General Purpose of Position */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">B. GENERAL PURPOSE OF POSITION</h2>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
              <div>
                <label htmlFor="divisionCollege" className="block text-sm font-medium text-gray-700 mb-1">Division or College:</label>
                <input
                  type="text"
                  id="divisionCollege"
                  name="divisionCollege"
                  value={formData.divisionCollege}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
              <div>
                <label htmlFor="department" className="block text-sm font-medium text-gray-700 mb-1">Department:</label>
                <input
                  type="text"
                  id="department"
                  name="department"
                  value={formData.department}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
              <div>
                <label htmlFor="jobCode" className="block text-sm font-medium text-gray-700 mb-1">Job Code:</label>
                <input
                  type="text"
                  id="jobCode"
                  name="jobCode"
                  value={formData.jobCode}
                  onChange={handleChange}
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>
              <div className="md:col-span-2">
                <label htmlFor="generalPurpose" className="block text-sm font-medium text-gray-700 mb-1">
                  Indicate in one or two sentences the general purpose of the position (or why this job exists).
                  This statement should be a general summary of the responsibilities listed in the next section.
                </label>
                <textarea
                  id="generalPurpose"
                  name="generalPurpose"
                  value={formData.generalPurpose}
                  onChange={handleChange}
                  rows="3"
                  className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                  required
                ></textarea>
              </div>
            </div>
          </section>

          {/* Section C: Summary of Responsibilities/Duties */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">C. SUMMARY OF RESPONSIBILITIES/DUTIES</h2>
            <p className="text-sm text-gray-600 mb-4">
              Describe specific job responsibilities/duties, listing the most important first. Use a separate statement for each
              responsibility. Most positions can be described in 6-8 major responsibility areas. Combine minor or occasional duties in
              one last statement. Give a best estimate of average percentage of time each responsibility takes; however, do not include
              a duty which occupies 5% or less of your time unless it is an essential part of the job. Each statement should be brief and
              concise, beginning with an action verb.
            </p>
            <div className="space-y-4">
              {formData.responsibilities.map((resp, index) => (
                <div key={index} className="grid grid-cols-12 gap-4 items-center">
                  <label htmlFor={`responsibilityDescription${index}`} className="col-span-1 text-sm font-medium text-gray-700">
                    {index + 1}.
                  </label>
                  <input
                    type="text"
                    id={`responsibilityDescription${index}`}
                    name={`responsibilityDescription${index}`}
                    value={resp.description}
                    onChange={handleChange}
                    placeholder={index === 8 ? "Perform other job-related duties as assigned." : "Describe responsibility..."}
                    className="col-span-9 px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                    readOnly={index === 8} // Make the last one read-only as it's a fixed statement
                  />
                  <div className="col-span-2 flex items-center">
                    <input
                      type="number"
                      id={`responsibilityPercentage${index}`}
                      name={`responsibilityPercentage${index}`}
                      value={resp.percentage}
                      onChange={handleChange}
                      placeholder="%"
                      className="w-full px-2 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                      min="0"
                      max="100"
                    />
                    <span className="ml-2 text-gray-600">%</span>
                  </div>
                </div>
              ))}
            </div>
          </section>

          {/* Section D: General Education & Experience - Education */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">D. EDUCATION</h2>
            <p className="text-sm text-gray-600 mb-4">
              Check the box that best indicates the minimum training/education requirements of this job. (Not necessarily your education, but the requirements for the job).
            </p>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {[
                "Up to 8 years of education",
                "9 to 11 years of education",
                "High School Diploma or GED",
                "Vocational/Technical/Business School",
                "Some College/Associate's Degree",
                "Bachelor's Degree",
                "Master's Degree",
                "Doctorate Degree"
              ].map((label, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`education-${index}`}
                    name="educationRequirement"
                    value={label}
                    checked={formData.educationRequirement === label}
                    onChange={handleChange}
                    className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                  />
                  <label htmlFor={`education-${index}`} className="ml-2 block text-sm text-gray-700">
                    {label}
                  </label>
                </div>
              ))}
            </div>
          </section>

          {/* Section E: General Education & Experience - Experience */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">E. EXPERIENCE</h2>
            <p className="text-sm text-gray-600 mb-4">
              TYPE OF EXPERIENCE NEEDED: Please indicate the specific job experience needed. For example, "accounting experience in an education environment" vs. "accounting experience". Be sure that the experience stated is what is actually required by the job, not what is preferred.
            </p>
            <textarea
              id="experienceType"
              name="experienceType"
              value={formData.experienceType}
              onChange={handleChange}
              rows="3"
              className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500 mb-6"
            ></textarea>

            <p className="text-sm text-gray-600 mb-4">
              Check the box which best indicates the minimum amount of experience described above. (Not necessarily your years of experience, but the requirements for the job.)
            </p>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
              {[
                "Less than 6 months",
                "6 months but less than 1 year",
                "1 year but less than 3 years",
                "3 but less than 5 years",
                "5 but less than 7 years",
                "7 years plus"
              ].map((label, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`experience-${index}`}
                    name="experienceAmount"
                    value={label}
                    checked={formData.experienceAmount === label}
                    onChange={handleChange}
                    className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                  />
                  <label htmlFor={`experience-${index}`} className="ml-2 block text-sm text-gray-700">
                    {label}
                  </label>
                </div>
              ))}
            </div>
          </section>

          {/* Section F: Type of Skills and/or Licensing/Certification Required */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">F. TYPE OF SKILLS AND/OR LICENSING/CERTIFICATION REQUIRED</h2>
            <p className="text-sm text-gray-600 mb-4">
              Please indicate all specific skills and/or licensing/certification required (not preferred) to do this job. For example, spreadsheet software proficiency may be a requirement for a secretarial job; journey license may be required for an electrician.
            </p>
            <textarea
              id="skillsCertifications"
              name="skillsCertifications"
              value={formData.skillsCertifications}
              onChange={handleChange}
              rows="4"
              className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
            ></textarea>
          </section>

          {/* Section G: Supervisory Responsibilities */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">G. SUPERVISORY RESPONSIBILITIES</h2>
            <p className="text-sm text-gray-600 mb-4">
              SUPERVISORY NATURE: What is the nature of the direct supervisory responsibility your job has? Check one answer.
            </p>
            <div className="grid grid-cols-1 md:grid-cols-2 gap-4 mb-6">
              {[
                "No supervisory responsibility.",
                "Work leadership of one or more employees.",
                "Supervisor over a section of a department.",
                "Assistant Manager over supervisors or a small department.",
                "Manager of one department.",
                "Manager of more than one department.",
                "Director, through managers, of a single department.",
                "Director, through managers, of multiple departments."
              ].map((label, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`supervisoryNature-${index}`}
                    name="supervisoryNature"
                    value={label}
                    checked={formData.supervisoryNature === label}
                    onChange={handleChange}
                    className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                  />
                  <label htmlFor={`supervisoryNature-${index}`} className="ml-2 block text-sm text-gray-700">
                    {label}
                  </label>
                </div>
              ))}
            </div>

            <p className="text-sm text-gray-600 mb-4">How many positions report directly to you?</p>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
              {["None", "1", "2-3", "4-6", "7 or more"].map((label, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`directReportsCount-${index}`}
                    name="directReportsCount"
                    value={label}
                    checked={formData.directReportsCount === label}
                    onChange={handleChange}
                    className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                  />
                  <label htmlFor={`directReportsCount-${index}`} className="ml-2 block text-sm text-gray-700">
                    {label}
                  </label>
                </div>
              ))}
            </div>

            <p className="text-sm font-medium text-gray-700 mb-2">List the title(s) of employee(s) whom you directly supervise:</p>
            <div className="border border-gray-300 rounded-md p-4 mb-6">
              <div className="grid grid-cols-6 gap-4 font-bold text-sm text-gray-800 mb-2">
                <span className="col-span-3">Title</span>
                <span className="col-span-2">Grade/Level</span>
                <span className="col-span-1">Number</span>
              </div>
              {formData.directReportTitles.map((dr, index) => (
                <div key={index} className="grid grid-cols-6 gap-4 mb-2">
                  <input
                    type="text"
                    name={`directReportTitle${index}`}
                    value={dr.title}
                    onChange={handleChange}
                    className="col-span-3 px-2 py-1 border border-gray-200 rounded-md shadow-sm text-sm"
                  />
                  <input
                    type="text"
                    name={`directReportGrade${index}`}
                    value={dr.gradeLevel}
                    onChange={handleChange}
                    className="col-span-2 px-2 py-1 border border-gray-200 rounded-md shadow-sm text-sm"
                  />
                  <input
                    type="number"
                    name={`directReportNum${index}`}
                    value={dr.numPositions}
                    onChange={handleChange}
                    className="col-span-1 px-2 py-1 border border-gray-200 rounded-md shadow-sm text-sm"
                    min="0"
                  />
                </div>
              ))}
            </div>

            <p className="text-sm text-gray-600 mb-4">Indicate the total number of employees you indirectly supervise through supervisors or managers:</p>
            <div className="grid grid-cols-2 md:grid-cols-4 gap-4 mb-6">
              {["None", "1-5", "6-10", "11-20", "21-50", "51-100", "100+"].map((label, index) => (
                <div key={index} className="flex items-center">
                  <input
                    type="radio"
                    id={`indirectReportsCount-${index}`}
                    name="indirectReportsCount"
                    value={label}
                    checked={formData.indirectReportsCount === label}
                    onChange={handleChange}
                    className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                  />
                  <label htmlFor={`indirectReportsCount-${index}`} className="ml-2 block text-sm text-gray-700">
                    {label}
                  </label>
                </div>
              ))}
            </div>

            <p className="text-sm text-gray-600 mb-4">Does this position require functional supervision of positions that do not report directly to you?</p>
            <div className="flex space-x-6">
              <div className="flex items-center">
                <input
                  type="radio"
                  id="functionalSupervision-yes"
                  name="functionalSupervision"
                  value="Yes"
                  checked={formData.functionalSupervision === "Yes"}
                  onChange={handleChange}
                  className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                />
                <label htmlFor="functionalSupervision-yes" className="ml-2 block text-sm text-gray-700">
                  Yes
                </label>
              </div>
              <div className="flex items-center">
                <input
                  type="radio"
                  id="functionalSupervision-no"
                  name="functionalSupervision"
                  value="No"
                  checked={formData.functionalSupervision === "No"}
                  onChange={handleChange}
                  className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                />
                <label htmlFor="functionalSupervision-no" className="ml-2 block text-sm text-gray-700">
                  No
                </label>
              </div>
            </div>
          </section>

          {/* Organization Chart */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">Organization Chart</h2>
            <p className="text-sm text-gray-600 mb-4">Please complete the organization chart below:</p>

            <div className="flex flex-col items-center space-y-8 p-4 border border-gray-300 rounded-lg bg-white relative">
              {/* Top: Immediate Supervisor */}
              <div className="flex flex-col items-center">
                <label htmlFor="orgChartSupervisor" className="text-sm font-medium text-gray-700 mb-1">Title of Your Immediate Supervisor</label>
                <input
                  type="text"
                  id="orgChartSupervisor"
                  name="orgChartSupervisor"
                  value={formData.orgChartSupervisor}
                  onChange={handleChange}
                  className="w-64 px-4 py-2 border border-gray-300 rounded-md shadow-sm text-center focus:ring-indigo-500 focus:border-indigo-500"
                />
              </div>

              {/* Lines from Supervisor */}
              <div className="w-full h-8 border-b-2 border-l-2 border-r-2 border-gray-500 rounded-b-lg flex justify-around">
                <div className="w-px h-full bg-gray-500 translate-x-1/2"></div>
                <div className="w-px h-full bg-gray-500 translate-x-1/2"></div>
                <div className="w-px h-full bg-gray-500 translate-x-1/2"></div>
                <div className="w-px h-full bg-gray-500 translate-x-1/2"></div>
                <div className="w-px h-full bg-gray-500 translate-x-1/2"></div>
              </div>

              {/* Middle Row: Other jobs and Your Position */}
              <div className="flex w-full justify-around items-start">
                {/* Other jobs left */}
                <div className="flex flex-col items-center w-1/5 text-center">
                  <span className="text-xs text-gray-600 mb-1">Other jobs which report to your immediate supervisor</span>
                  <input type="text" name="orgChartOtherJobsImmediateSupervisor0" value={formData.orgChartOtherJobsImmediateSupervisor[0]} onChange={handleChange} className="w-32 px-2 py-1 border border-gray-300 rounded-md shadow-sm text-center text-sm mb-2" />
                  <input type="text" name="orgChartOtherJobsImmediateSupervisor1" value={formData.orgChartOtherJobsImmediateSupervisor[1]} onChange={handleChange} className="w-32 px-2 py-1 border border-gray-300 rounded-md shadow-sm text-center text-sm" />
                </div>
                {/* 3 empty boxes and Your Position */}
                {Array(2).fill(0).map((_, i) => (
                  <div key={`empty-box-${i}`} className="w-32 h-16 border border-gray-300 rounded-md bg-gray-50 shadow-sm flex items-center justify-center"></div>
                ))}
                <div className="w-32 h-16 border border-indigo-500 rounded-md shadow-md flex flex-col items-center justify-center bg-indigo-50">
                  <span className="text-sm font-semibold text-indigo-800">Your Position</span>
                  <input
                    type="text"
                    id="orgChartYourPosition"
                    name="orgChartYourPosition"
                    value={formData.orgChartYourPosition}
                    onChange={handleChange}
                    className="w-full text-center bg-transparent border-none text-xs px-1"
                    placeholder="Your Job Title"
                  />
                </div>
                {Array(2).fill(0).map((_, i) => (
                  <div key={`empty-box-right-${i}`} className="w-32 h-16 border border-gray-300 rounded-md bg-gray-50 shadow-sm flex items-center justify-center"></div>
                ))}
                {/* Other jobs right (duplicate for visual symmetry as per original PDF) */}
                <div className="flex flex-col items-center w-1/5 text-center">
                  <span className="text-xs text-gray-600 mb-1">Other jobs which report to your immediate supervisor</span>
                  <input type="text" name="orgChartOtherJobsImmediateSupervisor0" value={formData.orgChartOtherJobsImmediateSupervisor[0]} onChange={handleChange} className="w-32 px-2 py-1 border border-gray-300 rounded-md shadow-sm text-center text-sm mb-2" />
                  <input type="text" name="orgChartOtherJobsImmediateSupervisor1" value={formData.orgChartOtherJobsImmediateSupervisor[1]} onChange={handleChange} className="w-32 px-2 py-1 border border-gray-300 rounded-md shadow-sm text-center text-sm" />
                </div>
              </div>

              {/* Lines from Your Position (conceptual, representing direct reports) */}
              <div className="w-32 h-8 border-b-2 border-gray-500 rounded-b-lg flex justify-around mt-4">
                <div className="w-px h-full bg-gray-500"></div>
                <div className="w-px h-full bg-gray-500"></div>
                <div className="w-px h-full bg-gray-500"></div>
              </div>


              {/* Bottom Row: Titles of Your Direct Reports */}
              <div className="flex w-full justify-around flex-wrap gap-4 mt-4">
                {formData.orgChartDirectReports.map((report, index) => (
                  <input
                    key={`direct-report-${index}`}
                    type="text"
                    name={`orgChartDirectReports${index}`}
                    value={report}
                    onChange={handleChange}
                    className="w-32 px-2 py-1 border border-gray-300 rounded-md shadow-sm text-center text-sm"
                    placeholder={`Direct Report ${index + 1}`}
                  />
                ))}
              </div>
            </div>
          </section>

          {/* Section H: Physical Demands and Working Conditions */}
          <section className="bg-blue-50 p-6 rounded-lg shadow-inner">
            <h2 className="text-2xl font-bold text-indigo-800 mb-4 border-b-2 border-indigo-400 pb-2">H. PHYSICAL DEMANDS AND WORKING CONDITIONS</h2>
            <p className="text-sm text-gray-600 mb-4">
              Indicate how often the following physical demands are required to perform the Essential Job Responsibilities.
            </p>
            <div className="grid grid-cols-2 gap-4 mb-6 text-sm text-gray-700">
              <p>C = Constantly (5-8 hrs./shift)</p>
              <p>F = Frequently (2-5 hrs./shift)</p>
              <p>O = Occasionally (Up to 2 hrs/shift)</p>
              <p>R = Rarely (Does not exist as regular part of job)</p>
            </div>

            <div className="grid grid-cols-1 md:grid-cols-2 gap-x-8 gap-y-4">
              {/* Physical Demands */}
              <div>
                <h3 className="font-semibold text-lg text-gray-800 mb-2">Physical Demands</h3>
                {[
                  "Standing", "Walking", "Sitting", "Lifting", "Carrying", "Pushing", "Pulling",
                  "Climbing", "Balancing", "Stooping", "Kneeling", "Crouching", "Crawling", "Reaching",
                  "Handling", "Grasping", "Feeling", "Talking", "Hearing", "Repetitive Motions", "Eye/Hand/Foot Coordination"
                ].map((demand, index) => (
                  <div key={index} className="flex justify-between items-center py-1 border-b border-gray-200 last:border-b-0">
                    <span className="text-gray-700">{demand}</span>
                    <div className="flex space-x-2">
                      {['C', 'F', 'O', 'R'].map(val => (
                        <label key={val} className="inline-flex items-center">
                          <input
                            type="radio"
                            name={`physicalDemands.${demand}`}
                            value={val}
                            checked={formData.physicalDemands[demand] === val}
                            onChange={(e) => handleCheckboxChange('physicalDemands', e)}
                            className="form-radio h-4 w-4 text-indigo-600"
                          />
                          <span className="ml-1 text-gray-600">{val}</span>
                        </label>
                      ))}
                    </div>
                  </div>
                ))}
              </div>

              {/* Environmental Conditions */}
              <div>
                <h3 className="font-semibold text-lg text-gray-800 mb-2">Environmental Conditions</h3>
                {[
                  "Extreme Cold", "Extreme Heat", "Temperature Changes", "Wet", "Humid", "Noise", "Vibration", "Hazards", "Atmospheric Conditions"
                ].map((condition, index) => (
                  <div key={index} className="flex justify-between items-center py-1 border-b border-gray-200 last:border-b-0">
                    <span className="text-gray-700">{condition}</span>
                    <div className="flex space-x-2">
                      {['C', 'F', 'O', 'R'].map(val => (
                        <label key={val} className="inline-flex items-center">
                          <input
                            type="radio"
                            name={`environmentalConditions.${condition}`}
                            value={val}
                            checked={formData.environmentalConditions[condition] === val}
                            onChange={(e) => handleCheckboxChange('environmentalConditions', e)}
                            className="form-radio h-4 w-4 text-indigo-600"
                          />
                          <span className="ml-1 text-gray-600">{val}</span>
                        </label>
                      ))}
                    </div>
                  </div>
                ))}
                <div className="mt-4">
                  <label htmlFor="otherEnvironmental" className="block text-sm font-medium text-gray-700 mb-1">Other (define):</label>
                  <input
                    type="text"
                    id="otherEnvironmental"
                    name="otherEnvironmental"
                    value={formData.environmentalConditions.otherEnvironmental || ''}
                    onChange={(e) => handleCheckboxChange('environmentalConditions', e)}
                    className="mt-1 block w-full px-4 py-2 border border-gray-300 rounded-md shadow-sm focus:ring-indigo-500 focus:border-indigo-500"
                  />
                </div>
              </div>
            </div>

            <div className="mt-6">
              <h3 className="font-semibold text-lg text-gray-800 mb-2">Physical Strength</h3>
              <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
                {["Little Physical Effort", "Light Work", "Medium Work", "Heavy Work", "Very Heavy Work"].map((strength, index) => (
                  <div key={index} className="flex items-center">
                    <input
                      type="radio"
                      id={`physicalStrength-${index}`}
                      name="physicalStrength"
                      value={strength}
                      checked={formData.physicalStrength === strength}
                      onChange={handleChange}
                      className="h-4 w-4 text-indigo-600 border-gray-300 focus:ring-indigo-500 rounded"
                    />
                    <label htmlFor={`physicalStrength-${index}`} className="ml-2 block text-sm text-gray-700">
                      {strength}
                    </label>
                  </div>
                ))}
              </div>
            </div>
          </section>


          <button
            type="submit"
            className="w-full bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-3 px-6 rounded-lg shadow-lg transform transition duration-300 hover:scale-105 focus:outline-none focus:ring-4 focus:ring-indigo-500 focus:ring-opacity-50"
          >
            Submit Questionnaire
          </button>
        </form>

        {submissionStatus && (
          <div className={`mt-8 p-4 rounded-md ${submissionStatus.startsWith('Error') ? 'bg-red-100 text-red-700' : 'bg-green-100 text-green-700'} text-center`}>
            {submissionStatus}
          </div>
        )}

        {showConfirmation && (
          <div className="fixed inset-0 bg-gray-600 bg-opacity-75 flex items-center justify-center z-50 p-4">
            <div className="bg-white p-8 rounded-lg shadow-xl max-w-sm w-full text-center">
              <h3 className="text-xl font-semibold text-gray-900 mb-4">Confirm Submission</h3>
              <p className="text-gray-700 mb-6">Are you sure you want to submit the questionnaire?</p>
              <div className="flex justify-around space-x-4">
                <button
                  onClick={confirmSubmit}
                  className="flex-1 bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded-md shadow-md transition duration-200"
                >
                  Yes, Submit
                </button>
                <button
                  onClick={cancelSubmit}
                  className="flex-1 bg-red-600 hover:bg-red-700 text-white font-bold py-2 px-4 rounded-md shadow-md transition duration-200"
                >
                  Cancel
                </button>
              </div>
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

export default App;

