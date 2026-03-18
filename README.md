# Smart-medical-kito
Giving alert messageto the doctors ,family members and gaurdian
import pandas as pd
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.metrics.pairwise import cosine_similarity

# Sample jobs data (expand with CSV)
jobs = pd.DataFrame({
    'Job Title': ['Software Developer', 'Data Analyst', 'Web Developer'],
    'Key Skills': ['Python, SQL, ML', 'Excel, SQL, Visualization', 'HTML, CSS, JavaScript, React']
})

# User input example
user_skills = "Python, Data Analysis"  # Candidate skills

# TF-IDF vectorizer
tfidf = TfidfVectorizer(stop_words='english')
tfidf_matrix = tfidf.fit_transform(jobs['Key Skills'].tolist() + [user_skills])
similarity = cosine_similarity(tfidf_matrix[-1:], tfidf_matrix[:-1])[0]

# Matches
matches = pd.DataFrame({'Similarity': similarity, 'Job': jobs['Job Title'], 'Skills': jobs['Key Skills']})
matches = matches.sort_values('Similarity', ascending=False).head(3)

# Prep notes dict (expand)
prep_notes = {
    'Software Developer': 'Practice LeetCode DSA, system design; review Python OOP.',
    'Data Analyst': 'SQL queries, Tableau/PowerBI; case studies on cleaning data.',
    'Web Developer': 'Build portfolio with React projects; CSS Flexbox/Grid.'
}

print(matches)
for idx, row in matches.iterrows():
    job = row['Job']
    print(f"\n{job}: Score {row['Similarity']:.2f}")
    print(f"Prep Notes: {prep_notes.get(job, 'General resume tips.')}")
