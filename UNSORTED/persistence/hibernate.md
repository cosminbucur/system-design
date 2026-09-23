- flush

// Assume these 2 operations are not in a transaction.

entityDao.save(entity);
dependentEntityDao.getByJoinQuery(dependentEntity, entity);

// The second query could fail as it required data from first query to be persisted.
